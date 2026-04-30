# 46 — Prototypes and the Prototype Chain

## What is this?

Every object in JavaScript has a hidden internal link to another object called its **prototype**. When you try to access a property or method on an object and JS can't find it directly on that object, it automatically walks up this chain of prototype links — looking at the parent, then the parent's parent, and so on — until it either finds the property or hits `null`. This chain is called the **prototype chain**. Think of it like an employee asking their manager for a resource: if the manager doesn't have it, they ask their manager, all the way up to the CEO. If the CEO doesn't have it either, the answer is `undefined`.

## Why does it matter?

The prototype chain is the engine behind *all* inheritance in JavaScript. When you call `.toString()` on a number, `.map()` on an array, or `.hasOwnProperty()` on any object — you are using the prototype chain. ES6 classes, constructor functions, and `Object.create()` all rely on it. Without understanding prototypes, you cannot understand why JS inheritance works, why some bugs happen, or how to use advanced patterns confidently.

---

## Syntax

```js
// Every function automatically gets a .prototype object
function Animal(name) {
  this.name = name;               // own property — lives on the instance
}

Animal.prototype.speak = function () {   // shared method — lives on the prototype
  return `${this.name} makes a sound.`;
};

const dog = new Animal("Rex");

// Accessing the prototype chain manually:
console.log(Object.getPrototypeOf(dog));   // Animal.prototype
console.log(Object.getPrototypeOf(Animal.prototype)); // Object.prototype
console.log(Object.getPrototypeOf(Object.prototype)); // null — end of chain
```

---

## How it works — line by line

```js
function Animal(name) {
  this.name = name;
}
```
Defines a constructor. JS immediately creates `Animal.prototype` — an empty object — and sets `Animal.prototype.constructor = Animal`.

```js
Animal.prototype.speak = function () { ... };
```
Attaches `speak` to the *shared* prototype object, not to any individual instance.

```js
const dog = new Animal("Rex");
```
Creates a new object. Sets `dog.__proto__ = Animal.prototype`. Runs the constructor body so `dog.name = "Rex"` is set as an **own property**.

```js
dog.speak()
```
1. JS looks for `speak` directly on `dog` → not found.
2. JS follows `dog.__proto__` (= `Animal.prototype`) → found! Calls it.

---

## The full prototype chain visualised

```
dog
├── name: "Rex"           ← own property
└── [[Prototype]] ──────► Animal.prototype
                          ├── speak: function   ← found here on method lookup
                          └── [[Prototype]] ──► Object.prototype
                                                ├── hasOwnProperty: function
                                                ├── toString: function
                                                ├── valueOf: function
                                                └── [[Prototype]] ──► null
                                                                      (chain ends)
```

When you call `dog.hasOwnProperty("name")`, JS walks: `dog` → `Animal.prototype` → `Object.prototype` → found `hasOwnProperty` there.

---

## Example 1 — basic: reading vs walking the chain

```js
function Vehicle(type) {
  this.type = type;              // own property
}

Vehicle.prototype.describe = function () {
  return `This is a ${this.type}.`;
};

const car = new Vehicle("car");

// Own property — found immediately on the object
console.log(car.type);          // "car"

// Prototype method — found by walking the chain
console.log(car.describe());    // "This is a car."

// From Object.prototype — walked all the way up
console.log(car.toString());    // "[object Object]"

// Checking WHERE a property actually lives:
console.log(car.hasOwnProperty("type"));      // true  — own property
console.log(car.hasOwnProperty("describe"));  // false — lives on prototype
console.log("describe" in car);               // true  — 'in' checks the whole chain
```

---

## Example 2 — real world: shared methods save memory

```js
function User(username, email) {
  this.username = username;   // unique per instance
  this.email    = email;
}

// ONE copy of each method shared across ALL User instances:
User.prototype.getProfile = function () {
  return `${this.username} <${this.email}>`;
};

User.prototype.sendMessage = function (message) {
  return `[${this.username}]: ${message}`;
};

User.prototype.isAdmin = false;   // default flag for all users

const alice = new User("alice99", "alice@example.com");
const bob   = new User("bob42",   "bob@example.com");

console.log(alice.getProfile());           // "alice99 <alice@example.com>"
console.log(bob.sendMessage("Hello!"));    // "[bob42]: Hello!"

// The methods are literally the same function object:
console.log(alice.getProfile === bob.getProfile);  // true — shared, not duplicated

// Overriding a prototype property on a single instance (shadowing):
alice.isAdmin = true;  // sets an OWN property on alice only
console.log(alice.isAdmin);  // true  — found on alice's own properties first
console.log(bob.isAdmin);    // false — bob still reads from User.prototype
```

---

## Example 3 — Object.create() for manual prototype control

`Object.create(proto)` creates a new object whose `[[Prototype]]` is set to `proto`. No constructor function needed.

```js
const animalBlueprint = {
  breathe() {
    return `${this.name} is breathing.`;
  },
  eat(food) {
    return `${this.name} eats ${food}.`;
  }
};

// dog's prototype IS animalBlueprint
const dog = Object.create(animalBlueprint);
dog.name  = "Rex";
dog.breed = "Labrador";

console.log(dog.breathe());      // "Rex is breathing."  — from prototype
console.log(dog.eat("kibble"));  // "Rex eats kibble."   — from prototype
console.log(dog.name);           // "Rex"                — own property

// cat is also linked to the same blueprint
const cat = Object.create(animalBlueprint);
cat.name = "Whiskers";
console.log(cat.eat("fish"));    // "Whiskers eats fish."

// Check the chain
console.log(Object.getPrototypeOf(dog) === animalBlueprint);  // true
```

---

## Example 4 — prototype chain in built-in types

This shows you've been using the prototype chain every single day:

```js
const scores = [10, 20, 30];

// scores is an Array instance.
// scores.__proto__ === Array.prototype
// Array.prototype has: push, pop, map, filter, reduce, ...

// Walking the chain to find .map:
// scores → (not found) → Array.prototype → found! .map exists here
console.log(scores.map(x => x * 2));   // [20, 40, 60]

// Walking further for .hasOwnProperty:
// scores → Array.prototype → Object.prototype → found!
console.log(scores.hasOwnProperty(0));  // true (index 0 is an own property)

// You can actually ADD to Array.prototype (but NEVER do this in production):
// Array.prototype.sum = function() { return this.reduce((a, b) => a + b, 0); }
// scores.sum()  // 60
// This is called "monkey patching" and it breaks other people's code

const str = "hello";
// str.__proto__ === String.prototype
// String.prototype has: toUpperCase, trim, split, includes, ...
console.log(str.toUpperCase());   // "HELLO"  — from String.prototype
```

---

## Deep dive: `__proto__` vs `.prototype` — the most confused pair in JS

These two things sound similar but are completely different:

| | `.prototype` | `.__proto__` (or `[[Prototype]]`) |
|---|---|---|
| **Lives on** | Functions only | Every object |
| **What it is** | The object that will become the `[[Prototype]]` of instances | The actual prototype link for *this specific object* |
| **Set when** | When function is defined | When object is created |
| **Purpose** | Blueprint for instances | Lookup chain for property access |

```js
function Cat(name) { this.name = name; }

// .prototype — exists on the FUNCTION
console.log(typeof Cat.prototype);   // "object"
console.log(Cat.prototype);          // { constructor: Cat }

const whiskers = new Cat("Whiskers");

// .__proto__ — exists on the INSTANCE, points to Cat.prototype
console.log(whiskers.__proto__ === Cat.prototype);  // true

// The function itself also has a __proto__:
console.log(Cat.__proto__ === Function.prototype);  // true — Cat IS a function object

// NEVER use __proto__ in real code — use the official API:
const proto = Object.getPrototypeOf(whiskers);      // preferred
Object.setPrototypeOf(whiskers, someOtherProto);    // preferred (but still rare)
```

---

## Property shadowing

When you assign a property to an instance that has the same name as a prototype property, the instance's version **shadows** (hides) the prototype version:

```js
function Config() {}
Config.prototype.theme = "light";   // default theme for all configs

const userConfig = new Config();
console.log(userConfig.theme);      // "light" — read from prototype

userConfig.theme = "dark";          // creates an OWN property on userConfig
console.log(userConfig.theme);      // "dark"  — own property shadows prototype

const defaultConfig = new Config();
console.log(defaultConfig.theme);   // "light" — prototype unchanged

// To delete the shadow and reveal prototype value:
delete userConfig.theme;
console.log(userConfig.theme);      // "light" — falls back to prototype again
```

---

## Checking properties up and down the chain

```js
function Product(name, price) {
  this.name  = name;
  this.price = price;
}
Product.prototype.category = "general";

const laptop = new Product("ThinkPad", 1200);

// hasOwnProperty — only own (NOT inherited)
console.log(laptop.hasOwnProperty("name"));      // true
console.log(laptop.hasOwnProperty("category"));  // false

// 'in' operator — checks entire chain
console.log("name"     in laptop);  // true
console.log("category" in laptop);  // true
console.log("color"    in laptop);  // false

// Object.keys() — only own enumerable
console.log(Object.keys(laptop));   // ["name", "price"]

// for...in — iterates own + inherited enumerable
for (const key in laptop) {
  console.log(key);  // name, price, category
}

// Safe pattern to only process own properties in for...in:
for (const key in laptop) {
  if (laptop.hasOwnProperty(key)) {
    console.log(key);   // name, price  (skips category)
  }
}
```

---

## Tricky things

### 1. Mutating the prototype after instances are created — live effect

```js
function Gadget(name) { this.name = name; }

const phone = new Gadget("iPhone");

// Add a method to the prototype AFTER 'phone' was created
Gadget.prototype.ring = function () { return `${this.name} is ringing!`; };

// 'phone' immediately gets access to it — the chain is live
console.log(phone.ring());  // "iPhone is ringing!"
```

The chain lookup happens at **call time**, not at creation time. This is powerful but can also cause confusion.

### 2. Replacing the entire prototype breaks the constructor reference

```js
function Robot(model) { this.model = model; }

// BAD: replacing the prototype object wholesale
Robot.prototype = {
  describe() { return `Robot: ${this.model}`; }
  // constructor reference is now GONE
};

const r = new Robot("R2D2");
console.log(r.constructor === Robot);   // false!  broken
console.log(r.constructor === Object);  // true   (inherited from Object.prototype)

// GOOD: add methods one-by-one OR restore constructor
Robot.prototype = {
  constructor: Robot,   // manually restore it
  describe() { return `Robot: ${this.model}`; }
};
// OR better: just add methods individually:
// Robot.prototype.describe = function() { ... }
```

### 3. `Object.create(null)` — an object with NO prototype

```js
const cleanMap = Object.create(null);  // no prototype chain at all
cleanMap.name = "Alice";

console.log(cleanMap.toString);       // undefined — no Object.prototype methods!
console.log(cleanMap.hasOwnProperty); // undefined

// Useful when you want a pure dictionary with no prototype pollution:
// if ("toString" in cleanMap) — would be false, unlike normal objects
```

### 4. Performance: deep chains are slower

Accessing a property that requires walking 5 levels of the chain is technically slower than one found immediately on the object. In practice this matters only at massive scale, but it explains why flatter hierarchies are preferred.

---

## Common mistakes

### Mistake 1 — Putting mutable state on the prototype

```js
// WRONG — all instances share the same array
function Team(name) {
  this.name = name;
}
Team.prototype.members = [];   // shared mutable reference!

const teamA = new Team("Alpha");
const teamB = new Team("Beta");
teamA.members.push("Alice");
console.log(teamB.members);   // ["Alice"] — CONTAMINATED

// RIGHT — mutable state in the constructor
function Team(name) {
  this.name    = name;
  this.members = [];   // each instance gets its own array
}
```

### Mistake 2 — Using `__proto__` directly

```js
// WRONG — __proto__ is deprecated for direct use
obj.__proto__ = someProto;

// RIGHT — use the official API
Object.setPrototypeOf(obj, someProto);
const proto = Object.getPrototypeOf(obj);
```

### Mistake 3 — Checking `typeof` instead of `instanceof` for instances

```js
function Dog(name) { this.name = name; }
const rex = new Dog("Rex");

// WRONG for custom constructors
console.log(typeof rex);          // "object" — tells you nothing useful

// RIGHT
console.log(rex instanceof Dog);  // true
console.log(rex instanceof Object); // also true — everything is an Object
```

---

## Practice exercises

### Exercise 1 — easy

Create a constructor function `Shape` with a property `color`. Add a method on `Shape.prototype` called `describe()` that returns `"A [color] shape."`. Create two shapes with different colors and verify:
- Both can call `describe()`
- `describe` is the same function on both (use `===`)
- `color` is an own property but `describe` is not

```js
// Write your code here
```

### Exercise 2 — medium

Create a constructor `Animal` with properties `name` and `sound`. Add a prototype method `speak()` returning `"[name] says [sound]!"`. Then create a `Dog` constructor that also sets `name` and `sound`, and manually set `Dog.prototype` so that Dog instances inherit `speak()` from `Animal.prototype`. Create a `Dog` instance and verify:
- `speak()` works on the dog
- `dog instanceof Dog` is `true`
- `dog instanceof Animal` is `true`
- `Object.getPrototypeOf(Dog.prototype) === Animal.prototype` is `true`

(Hint: use `Object.create(Animal.prototype)` to set `Dog.prototype`)

```js
// Write your code here
```

### Exercise 3 — hard

Build a prototype chain manually using only `Object.create()` — no constructor functions, no classes.

Create three objects:
1. `livingThing` — has a method `breathe()` returning `"[name] breathes."`
2. `animal` — its prototype is `livingThing`. Has a method `eat(food)` returning `"[name] eats [food]."`
3. `dog` — its prototype is `animal`. Has own properties `name: "Rex"` and `breed: "Labrador"`. Has a method `bark()` returning `"Rex barks!"`.

Then verify the full chain:
- `dog.bark()` works (own method)
- `dog.eat("meat")` works (from animal)
- `dog.breathe()` works (from livingThing)
- `Object.getPrototypeOf(dog) === animal` is `true`
- `Object.getPrototypeOf(animal) === livingThing` is `true`
- `dog.hasOwnProperty("breathe")` is `false`

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| `[[Prototype]]` | Hidden internal link every object has to its parent object |
| `Object.getPrototypeOf(obj)` | Official way to read an object's prototype |
| `fn.prototype` | The object that becomes `[[Prototype]]` of instances created by `new fn()` |
| `obj.__proto__` | Deprecated accessor for `[[Prototype]]` — use `getPrototypeOf` instead |
| Property lookup | Check own → walk `[[Prototype]]` → … → `Object.prototype` → `null` |
| `hasOwnProperty(key)` | `true` only if key is directly on the object, not inherited |
| `key in obj` | `true` if key exists anywhere in the chain |
| `obj instanceof Fn` | `true` if `Fn.prototype` is anywhere in obj's prototype chain |
| `Object.create(proto)` | Creates object whose `[[Prototype]]` is set to `proto` |
| `Object.create(null)` | Creates object with NO prototype chain |
| Property shadowing | Own property with same name as prototype property hides the prototype one |
| Prototype mutation | Changes to `fn.prototype` affect all existing instances immediately |
| `constructor` property | `Fn.prototype.constructor === Fn` — breaks if you replace `Fn.prototype` wholesale |
| Built-in chains | Arrays: `arr → Array.prototype → Object.prototype → null` |

---

## Connected topics

- **45 — Constructor functions** — Shows where `fn.prototype` comes from and how `new` links instances to it.
- **47 — ES6 Classes** — Classes are syntactic sugar over exactly this: a constructor function + methods on the prototype. Knowing this topic makes classes completely transparent.
- **48 — Inheritance** — Class `extends` works by wiring up the prototype chain between two constructors. This topic is the prerequisite.
