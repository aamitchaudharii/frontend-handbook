# 06 — Prototypes & Prototype Chain

> **"In JavaScript, inheritance is not about classes copying behavior — it's about objects delegating to other objects. The prototype chain is that delegation system."**

Prototypes are the foundation of everything in JavaScript. Every object, array, function, and class uses the prototype system under the hood. Understanding it deeply means you understand how property lookup works, how `class` syntax really works, why `instanceof` behaves the way it does, and how to design efficient object hierarchies.

---

## 📚 Table of Contents

1. [The Core Idea — Delegation, Not Copying](#1-the-core-idea--delegation-not-copying)
2. [Every Object Has a Prototype](#2-every-object-has-a-prototype)
3. [The Prototype Chain — Property Lookup](#3-the-prototype-chain--property-lookup)
4. [`[[Prototype]]` vs `.prototype`](#4-prototype-vs-prototype)
5. [Constructor Functions and `new`](#5-constructor-functions-and-new)
6. [The `class` Syntax — Syntactic Sugar](#6-the-class-syntax--syntactic-sugar)
7. [Prototype Methods vs Instance Methods](#7-prototype-methods-vs-instance-methods)
8. [Inheritance Patterns](#8-inheritance-patterns)
9. [Object.create — Prototype Without Constructor](#9-objectcreate--prototype-without-constructor)
10. [Built-in Prototypes](#10-built-in-prototypes)
11. [Property Shadowing](#11-property-shadowing)
12. [Performance Implications](#12-performance-implications)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. The Core Idea — Delegation, Not Copying

Classical class-based inheritance (Java, C++) works by **copying**: when you create a subclass, the methods are copied (or linked through vtables) into the child class.

JavaScript prototypes work by **delegation**: when you look up a property on an object, if it's not found there, the lookup is _delegated_ to another object — the prototype. This chain continues until the property is found or the chain ends at `null`.

```
Classical Inheritance (copy):
  Parent ──copies──► Child ──copies──► GrandChild

  Each level has its own copy of the methods.

Prototypal Delegation (link):
  instance ──delegates to──► Parent.prototype ──delegates to──► Object.prototype ──► null

  Methods live in ONE place. Instances just hold a reference.
```

This has a huge practical implication: **one million objects sharing the same prototype method consume memory for only one copy of that method**.

---

## 2. Every Object Has a Prototype

Almost every object in JavaScript has a prototype — an object it delegates to for property lookups.

```javascript
const obj = { name: "Alice" };

// Access the prototype:
Object.getPrototypeOf(obj); // Object.prototype
// Or via the deprecated (but widely supported) __proto__:
obj.__proto__ === Object.prototype; // true
```

```
obj:
┌────────────────────────┐
│  name: 'Alice'         │
│  [[Prototype]]: ───────┼──► Object.prototype
└────────────────────────┘       │
                                  │  hasOwnProperty()
                                  │  toString()
                                  │  valueOf()
                                  │  [[Prototype]]: null
                                  └────────────────────
```

### The Prototype of Different Types

```javascript
// Objects → Object.prototype
const obj = {};
Object.getPrototypeOf(obj) === Object.prototype; // true

// Arrays → Array.prototype → Object.prototype
const arr = [];
Object.getPrototypeOf(arr) === Array.prototype; // true
Object.getPrototypeOf(Array.prototype) === Object.prototype; // true

// Functions → Function.prototype → Object.prototype
const fn = () => {};
Object.getPrototypeOf(fn) === Function.prototype; // true

// Strings (boxed) → String.prototype → Object.prototype
Object.getPrototypeOf(String.prototype) === Object.prototype; // true

// The end of every chain:
Object.getPrototypeOf(Object.prototype); // null — chain ends here
```

---

## 3. The Prototype Chain — Property Lookup

When you access a property on an object, JavaScript performs the following lookup:

```
property lookup for: obj.someProperty

1. Does obj have own property 'someProperty'?
   → Yes: return it
   → No: continue

2. Does obj's [[Prototype]] have 'someProperty'?
   → Yes: return it
   → No: continue

3. Does [[Prototype]]'s [[Prototype]] have 'someProperty'?
   → Yes: return it
   → No: continue

4. Continue up the chain...

5. Reached [[Prototype]] = null?
   → Return undefined
```

### Visual Example

```javascript
const animal = {
  breathe() {
    return "breathing";
  },
};

const dog = {
  bark() {
    return "woof";
  },
};
Object.setPrototypeOf(dog, animal); // dog → animal

const rex = {
  name: "Rex",
};
Object.setPrototypeOf(rex, dog); // rex → dog → animal

rex.name; // 'Rex'      — found on rex itself
rex.bark(); // 'woof'     — not on rex, found on dog (one level up)
rex.breathe(); // 'breathing' — not on rex or dog, found on animal (two levels up)
rex.fly; // undefined  — not found anywhere in chain
```

```
Chain visualization:

rex: { name: 'Rex' }
  [[Prototype]] ──► dog: { bark() }
                      [[Prototype]] ──► animal: { breathe() }
                                          [[Prototype]] ──► Object.prototype
                                                              [[Prototype]] ──► null
```

### `hasOwnProperty` vs Inherited

```javascript
console.log(rex.hasOwnProperty("name")); // true  — own property
console.log(rex.hasOwnProperty("bark")); // false — inherited
console.log("bark" in rex); // true  — in operator checks chain
console.log("fly" in rex); // false — not in chain either
```

Use `hasOwnProperty` (or `Object.hasOwn(obj, key)` in modern JS) when you need to distinguish own vs inherited properties.

---

## 4. `[[Prototype]]` vs `.prototype`

This is the most common source of confusion about the prototype system. They are **different things**.

### `[[Prototype]]` (Internal Slot)

- Every **object** has `[[Prototype]]`
- It's the object this object delegates to for property lookups
- Accessed via `Object.getPrototypeOf(obj)` or `obj.__proto__` (deprecated)
- This is what forms the prototype chain

### `.prototype` (Property on Functions)

- Only **functions** have a `.prototype` property (not all objects)
- It's an object that becomes the `[[Prototype]]` of objects created with `new FunctionName()`
- It's a regular object property — not the function's own prototype
- `Function.prototype` is the function's own `[[Prototype]]`

```javascript
function Dog(name) {
  this.name = name;
}

// Dog.prototype — the object that will be [[Prototype]] of instances
Dog.prototype.bark = function () {
  return "woof";
};

const rex = new Dog("Rex");

// rex's [[Prototype]] IS Dog.prototype
Object.getPrototypeOf(rex) === Dog.prototype; // true ✓

// Dog's own [[Prototype]] is Function.prototype (Dog IS a function)
Object.getPrototypeOf(Dog) === Function.prototype; // true ✓

// Dog.prototype is NOT Dog's [[Prototype]] — it's a regular property
Dog.prototype !== Object.getPrototypeOf(Dog); // true — they're different things
```

```
VISUAL DISTINCTION:

Dog (function object):
  .prototype: ──────────────────────────► Dog.prototype object
  [[Prototype]]: ──────────────────────► Function.prototype
                                          (Dog is a function,
                                           so it delegates to Function.prototype)

rex (instance):
  .name: 'Rex'
  [[Prototype]]: ──────────────────────► Dog.prototype object
                                          .bark: function() { return 'woof'; }
                                          .constructor: Dog
                                          [[Prototype]]: Object.prototype
```

### The `.constructor` Property

`Dog.prototype.constructor` points back to `Dog`. This is how `instanceof` works and how you can determine an object's constructor:

```javascript
function Dog(name) {
  this.name = name;
}

const rex = new Dog("Rex");
rex.constructor === Dog; // true — inherited from Dog.prototype.constructor

// WARNING: this can be wrong if prototype is replaced
Dog.prototype = { bark() {} }; // replaces entire prototype
// constructor is now gone from the new prototype
rex2.constructor === Dog; // false — constructor is now Object (from Object.prototype)

// Fix: always restore constructor when replacing prototype
Dog.prototype = {
  constructor: Dog, // explicitly restore
  bark() {},
};
```

---

## 5. Constructor Functions and `new`

Before `class` syntax, constructor functions were how you created object families in JavaScript.

### What `new` Does — Step by Step

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.greet = function () {
  return `Hi, I'm ${this.name}`;
};

const alice = new Person("Alice", 30);
```

`new Person('Alice', 30)` does exactly 4 things:

```
Step 1: Create a new empty object
  newObj = Object.create(Person.prototype)
  // newObj's [[Prototype]] = Person.prototype

Step 2: Set this = newObj inside Person
  // this refers to the freshly created object

Step 3: Execute Person's body with this = newObj
  this.name = 'Alice'; // newObj.name = 'Alice'
  this.age  = 30;      // newObj.age  = 30

Step 4: Return newObj (unless Person explicitly returns an object)
  alice = newObj
```

### Manual Implementation of `new`

```javascript
function myNew(Constructor, ...args) {
  // Step 1: create object with Constructor.prototype as prototype
  const obj = Object.create(Constructor.prototype);

  // Step 2 & 3: call constructor with obj as this
  const result = Constructor.apply(obj, args);

  // Step 4: return obj unless constructor explicitly returned an object
  return result && typeof result === "object" ? result : obj;
}

const alice = myNew(Person, "Alice", 30);
alice.greet(); // "Hi, I'm Alice" ✓
alice instanceof Person; // true ✓
```

### When the Constructor Returns an Object

```javascript
function Weird() {
  this.x = 1;
  return { x: 99 }; // explicit object return
}

const w = new Weird();
w.x; // 99 — constructor's returned object wins
w instanceof Weird; // false — w is the returned {}, not the new object

function NotWeird() {
  this.x = 1;
  return 42; // primitive return — ignored
}

const nw = new NotWeird();
nw.x; // 1 — primitive return ignored, returns this
```

---

## 6. The `class` Syntax — Syntactic Sugar

`class` is syntactic sugar over prototype-based inheritance. It produces the same prototype chain — it just has a cleaner syntax.

```javascript
// ES6 class syntax:
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} makes a sound.`;
  }
}

class Dog extends Animal {
  constructor(name) {
    super(name); // calls Animal constructor
  }

  speak() {
    return `${this.name} barks.`;
  }
}
```

### What the Engine Actually Creates

```javascript
// The class above is equivalent to:

function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function () {
  return `${this.name} makes a sound.`;
};

function Dog(name) {
  Animal.call(this, name); // super(name)
}
// Set up prototype chain: Dog instances → Animal.prototype
Object.setPrototypeOf(Dog.prototype, Animal.prototype);
// Also set up static chain: Dog → Animal (for static methods)
Object.setPrototypeOf(Dog, Animal);

Dog.prototype.speak = function () {
  return `${this.name} barks.`;
};
```

### Key Differences: `class` vs Constructor Functions

| Feature        | Constructor Function            | `class`                    |
| -------------- | ------------------------------- | -------------------------- |
| Syntax         | Older, verbose                  | Modern, clean              |
| Hoisting       | Hoisted (as undefined function) | Not hoisted (TDZ)          |
| Strict mode    | Optional                        | Always strict              |
| `new` required | No (silently broken)            | Yes (throws without `new`) |
| Static methods | Manual on function              | `static` keyword           |
| Private fields | Convention (`_name`)            | True private (`#name`)     |
| `super`        | Manual (`Parent.call`)          | Built-in keyword           |

### Private Fields (True Encapsulation)

```javascript
class BankAccount {
  #balance = 0; // truly private — not accessible outside class
  #owner;

  constructor(owner, initialBalance) {
    this.#owner = owner;
    this.#balance = initialBalance;
  }

  deposit(amount) {
    if (amount <= 0) throw new Error("Invalid amount");
    this.#balance += amount;
    return this;
  }

  get balance() {
    return this.#balance;
  }
}

const acc = new BankAccount("Alice", 1000);
acc.deposit(500).deposit(200);
console.log(acc.balance); // 1700

acc.#balance; // SyntaxError — truly private, not just convention
```

---

## 7. Prototype Methods vs Instance Methods

This is a critical performance distinction.

### Instance Methods (per-object copies)

```javascript
function Dog(name) {
  this.name = name;
  // ❌ New function created for EVERY instance
  this.bark = function () {
    return `${this.name} says woof`;
  };
}

const d1 = new Dog("Rex");
const d2 = new Dog("Fido");

d1.bark === d2.bark; // false — DIFFERENT function objects
// 1000 Dog instances = 1000 separate bark functions in memory
```

### Prototype Methods (shared across all instances)

```javascript
function Dog(name) {
  this.name = name;
  // name is per-instance: stored on the object itself
}

// ✅ bark defined ONCE on the prototype — shared by all instances
Dog.prototype.bark = function () {
  return `${this.name} says woof`;
};

const d1 = new Dog("Rex");
const d2 = new Dog("Fido");

d1.bark === d2.bark; // true — SAME function object
// 1000 Dog instances = 1 bark function in memory
```

### Memory Impact

```javascript
// With instance methods:
const dogs = Array.from({ length: 10000 }, (_, i) => new Dog(`dog${i}`));
// 10,000 function objects in memory — one per instance

// With prototype methods:
const dogs = Array.from({ length: 10000 }, (_, i) => new Dog(`dog${i}`));
// 1 function object in memory — shared via prototype
```

### Rule of Thumb

- **Instance properties** (data unique per object) → on `this` in constructor
- **Methods** (shared behavior) → on `prototype`
- **Private state** → closure or `#privateField`

---

## 8. Inheritance Patterns

### Classical Inheritance with Constructor Functions

```javascript
function Shape(color) {
  this.color = color;
}

Shape.prototype.describe = function () {
  return `A ${this.color} shape`;
};

function Circle(color, radius) {
  Shape.call(this, color); // inherit properties
  this.radius = radius;
}

// Set up prototype chain — Circle instances delegate to Shape.prototype
Circle.prototype = Object.create(Shape.prototype);
Circle.prototype.constructor = Circle; // restore constructor

Circle.prototype.area = function () {
  return Math.PI * this.radius ** 2;
};

const c = new Circle("red", 5);
c.describe(); // 'A red shape' — inherited from Shape.prototype
c.area(); // 78.54...
c instanceof Circle; // true
c instanceof Shape; // true — chain includes Shape.prototype
```

### `class` Inheritance (Modern)

```javascript
class Shape {
  constructor(color) {
    this.color = color;
  }
  describe() {
    return `A ${this.color} shape`;
  }
}

class Circle extends Shape {
  constructor(color, radius) {
    super(color); // must call super before accessing this
    this.radius = radius;
  }
  area() {
    return Math.PI * this.radius ** 2;
  }
  // Override parent method
  describe() {
    return `${super.describe()} (circle, r=${this.radius})`;
  }
}

const c = new Circle("red", 5);
c.describe(); // 'A red shape (circle, r=5)'
```

### Mixin Pattern (Multiple Behavior Sources)

JavaScript only has single prototype inheritance. Mixins allow composing behavior from multiple sources:

```javascript
// Define behaviors as plain objects or functions
const Serializable = {
  serialize() {
    return JSON.stringify(this);
  },
  deserialize(s) {
    return Object.assign(this, JSON.parse(s));
  },
};

const Validatable = {
  validate() {
    return Object.keys(this.rules || {}).every((key) =>
      this.rules[key](this[key]),
    );
  },
};

// Mix them in
class User {
  constructor(data) {
    Object.assign(this, data);
  }
}

// Apply mixins
Object.assign(User.prototype, Serializable, Validatable);

const user = new User({ name: "Alice", email: "alice@example.com" });
user.serialize(); // '{"name":"Alice","email":"alice@example.com"}'
```

---

## 9. Object.create — Prototype Without Constructor

`Object.create(proto)` creates a new object with `proto` as its `[[Prototype]]`, without a constructor function:

```javascript
const personProto = {
  greet() {
    return `Hi, I'm ${this.name}`;
  },
  toString() {
    return `Person(${this.name})`;
  },
};

// Create person objects with personProto as prototype
const alice = Object.create(personProto);
alice.name = "Alice";
alice.age = 30;

alice.greet(); // 'Hi, I'm Alice' — delegated to personProto

// Verify the chain
Object.getPrototypeOf(alice) === personProto; // true
```

### Using `Object.create(null)` for Dict-like Objects

```javascript
// ❌ Regular object has prototype — inherited properties can interfere
const cache = {};
cache["toString"] = "value"; // shadows Object.prototype.toString
cache.hasOwnProperty; // a method from prototype — unexpected

// ✅ Object.create(null) — truly empty, no prototype
const cache = Object.create(null);
cache["toString"] = "value"; // just a string, no shadowing issues
cache.hasOwnProperty; // undefined — no prototype at all
"toString" in cache; // true — only own properties

// Perfect for: dictionaries, caches, maps without symbol pollution
```

---

## 10. Built-in Prototypes

All built-in types use the prototype system. Understanding this explains many JavaScript behaviors.

```javascript
// Array methods come from Array.prototype
const arr = [1, 2, 3];
arr.map; // function — inherited from Array.prototype
Object.getPrototypeOf(arr) === Array.prototype; // true

// Array.prototype itself delegates to Object.prototype
Object.getPrototypeOf(Array.prototype) === Object.prototype; // true

// Full chain for an array:
// arr → Array.prototype → Object.prototype → null
```

### Extending Built-in Prototypes — Why Not To

```javascript
// ❌ Prototype pollution — affects ALL arrays in the codebase
Array.prototype.sum = function () {
  return this.reduce((a, b) => a + b, 0);
};

// Problems:
// 1. Breaks for...in loops on arrays
// 2. Conflicts with future native implementations
// 3. Conflicts with other libraries doing the same
// 4. Makes code hard to understand — unexpected methods on builtins

// ✅ Standalone utility function instead
function sumArray(arr) {
  return arr.reduce((a, b) => a + b, 0);
}
```

---

## 11. Property Shadowing

When an own property has the same name as a prototype property, the own property **shadows** the prototype property.

```javascript
function Animal(name) {
  this.name = name;
}
Animal.prototype.type = "generic";

const dog = new Animal("Rex");
dog.type; // 'generic' — from prototype

dog.type = "canine"; // creates OWN property on dog
dog.type; // 'canine' — own property shadows prototype
Animal.prototype.type; // 'generic' — prototype unchanged

// The prototype's `type` is not modified — dog.type is a new own property
```

### Shadowing Methods — Overriding

```javascript
class Base {
  greet() {
    return "Hello from Base";
  }
}

class Child extends Base {
  greet() {
    return "Hello from Child";
  } // shadows Base.prototype.greet
}

const c = new Child();
c.greet(); // 'Hello from Child' — own prototype's method found first

// Access the shadowed method via super (in class) or directly:
Base.prototype.greet.call(c); // 'Hello from Base' — explicit access
```

### Non-Writable Property Shadowing Silently Fails

```javascript
const proto = {};
Object.defineProperty(proto, "x", {
  value: 1,
  writable: false,
});

const obj = Object.create(proto);
obj.x = 2; // Silently fails in non-strict mode
// Throws TypeError in strict mode
console.log(obj.x); // 1 — shadow not created
```

---

## 12. Performance Implications

### Prototype Lookup Cost

Property lookups that require traversing the prototype chain are slightly slower than own-property lookups. The difference is small in modern V8 (inline caches optimize repeated lookups), but it's measurable for extremely hot paths.

```javascript
// V8's inline cache:
// First access of obj.method:  traverse chain → cache the lookup
// Subsequent accesses:         use cached result (effectively O(1))
// As long as the prototype chain shape doesn't change
```

### Shape Stability and Hidden Classes

V8 uses "hidden classes" (shapes) to optimize object property access. Objects with the same shape (same properties in same order) share an optimized internal representation.

```javascript
// ✅ Consistent shape — V8 can optimize
function Point(x, y) {
  this.x = x; // always set in same order
  this.y = y;
}
const points = Array.from({ length: 10000 }, (_, i) => new Point(i, i));
// All points share the same hidden class → highly optimized

// ❌ Inconsistent shape — V8 creates multiple hidden classes
function makeObj(hasExtra) {
  const obj = { x: 1, y: 2 };
  if (hasExtra) obj.z = 3; // sometimes z exists, sometimes not
  return obj;
}
// Two hidden classes: {x,y} and {x,y,z}
// V8 can't as efficiently optimize code that handles both
```

### Prototype Chain Depth

Keep prototype chains shallow. Each link adds a lookup step.

```javascript
// ❌ Deep chain — 8 levels of delegation
class A {}
class B extends A {}
class C extends B {}
class D extends C {}
class E extends D {}
class F extends E {}
class G extends F {}
class H extends G {} // 8 levels deep

// Accessing a property only on A.prototype requires 8 lookups
// (mitigated by inline caches on repeated access)

// ✅ Prefer composition over deep inheritance
class Component {
  constructor() {
    this.renderer = new Renderer();
    this.eventBus = new EventBus();
    // compose behavior instead of extending
  }
}
```

---

## 13. Good Practices

### ✅ Use `class` for object families

```javascript
// ✅ Clear, safe, strict mode by default
class EventEmitter {
  #listeners = new Map();

  on(event, handler) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(handler);
    return () => this.off(event, handler);
  }

  off(event, handler) {
    this.#listeners.get(event)?.delete(handler);
  }

  emit(event, ...args) {
    this.#listeners.get(event)?.forEach((h) => h(...args));
  }
}
```

### ✅ Use `Object.create(null)` for pure dictionaries

```javascript
const cache = Object.create(null); // no inherited properties
```

### ✅ Check own properties explicitly when iterating

```javascript
const obj = { a: 1, b: 2 };

// ✅ for...of Object.keys — only own enumerable properties
for (const key of Object.keys(obj)) {
  console.log(key, obj[key]);
}

// ✅ Object.hasOwn (modern) — safer than hasOwnProperty
if (Object.hasOwn(obj, "a")) {
  /* ... */
}

// ⚠️ for...in — iterates inherited enumerable properties too
for (const key in obj) {
  if (Object.hasOwn(obj, key)) {
    // guard needed
    console.log(key);
  }
}
```

### ✅ Prefer composition over deep inheritance

```javascript
// ✅ Compose behavior via mixins or owned objects
class Dashboard {
  constructor() {
    this.dataManager = new DataManager();
    this.renderer = new Renderer();
    this.eventBus = new EventBus();
  }
}
// Flat, clear dependencies. No 8-level chain.
```

---

## 14. Bad Practices

### ❌ Extending built-in prototypes

```javascript
// ❌ Never do this in library or application code
Array.prototype.sum = function () {
  /* ... */
};
Object.prototype.deepClone = function () {
  /* ... */
};
String.prototype.capitalize = function () {
  /* ... */
};
```

### ❌ Replacing prototype entirely (breaks constructor)

```javascript
// ❌ Loses constructor reference
function Dog(name) {
  this.name = name;
}
Dog.prototype = {
  bark() {
    return "woof";
  },
  // constructor is now Object, not Dog!
};

// ✅ Always restore constructor when replacing
Dog.prototype = {
  constructor: Dog, // explicitly set
  bark() {
    return "woof";
  },
};
```

### ❌ Using `__proto__` directly

```javascript
// ❌ Deprecated — use Object.setPrototypeOf instead
obj.__proto__ = anotherObj;

// ✅
Object.setPrototypeOf(obj, anotherObj);
// Or better: use Object.create at creation time
```

### ❌ Deep inheritance for behavior reuse

```javascript
// ❌ 6-level deep chain for behavior that could be composed
class Widget extends UIElement extends Component
  extends Renderable extends Observable extends Base { }
```

---

## 15. Common Mistakes

### Mistake 1 — Confusing `[[Prototype]]` with `.prototype`

```javascript
function Foo() {}
const f = new Foo();

// These are different:
f.__proto__ === Foo.prototype; // true — f's [[Prototype]] IS Foo.prototype
Foo.__proto__ === Function.prototype; // true — Foo's [[Prototype]] is Function.prototype
Foo.prototype !== Foo.__proto__; // they're different objects!
```

### Mistake 2 — Defining methods in the constructor

```javascript
// ❌ New function per instance — memory waste
function Dog(name) {
  this.name = name;
  this.bark = function () {
    return "woof";
  }; // bad!
}

// ✅ Define once on prototype
Dog.prototype.bark = function () {
  return "woof";
};
```

### Mistake 3 — `instanceof` across different realms

```javascript
// In iframes or Node.js VMs, each realm has its own Object.prototype
// instanceof fails across realms
const iframe = document.createElement("iframe");
document.body.appendChild(iframe);
const iframeArray = new iframe.contentWindow.Array();

iframeArray instanceof Array; // false — different Array prototype!

// ✅ Use Array.isArray — works across realms
Array.isArray(iframeArray); // true
```

### Mistake 4 — Mutating shared prototype state

```javascript
function Team() {
  // ❌ members is on the prototype — SHARED across all Team instances
  // Adding to one instance's members adds to ALL instances
}
Team.prototype.members = []; // BAD: shared mutable state

const t1 = new Team();
const t2 = new Team();
t1.members.push("Alice");
console.log(t2.members); // ['Alice'] — unexpected! shared array

// ✅ Initialize mutable state in constructor
function Team() {
  this.members = []; // own property per instance
}
```

---

## 16. Interview-Level Explanation

> **"How does prototypal inheritance work in JavaScript? What's the difference between `[[Prototype]]` and `.prototype`?"**

**Strong answer:**

> "JavaScript uses prototypal delegation rather than classical inheritance. Every object has an internal `[[Prototype]]` slot — a reference to another object it delegates to when a property isn't found on the object itself. This forms a chain — the prototype chain — that terminates at `Object.prototype`, whose own `[[Prototype]]` is null.
>
> When you access a property, the engine first checks the object's own properties. If not found, it follows the `[[Prototype]]` reference to the next object and checks there, continuing up the chain until either the property is found or the chain ends at null, returning undefined.
>
> The distinction between `[[Prototype]]` and `.prototype` is important. `[[Prototype]]` is the internal slot on every object that forms the delegation chain. `.prototype` is a regular property that only exists on functions — it's the object that becomes the `[[Prototype]]` of instances created with `new ThatFunction()`. So `Dog.prototype` is what instances of Dog delegate to, but `Dog`'s own `[[Prototype]]` is `Function.prototype` because Dog itself is a function.
>
> The `class` syntax is syntactic sugar over this system. `class Dog extends Animal` sets up the prototype chain so Dog instances delegate to `Dog.prototype`, which itself delegates to `Animal.prototype`. The `extends` keyword also sets `Dog`'s `[[Prototype]]` to `Animal` for static method inheritance.
>
> A key performance point: methods should be on the prototype, not defined in the constructor. One million instances with prototype methods use memory for one function object. One million instances with constructor-defined methods use memory for one million function objects."

---

## 17. Exercises

### Exercise 1 — Trace the chain

```javascript
function Vehicle(type) {
  this.type = type;
}
Vehicle.prototype.describe = function () {
  return `I am a ${this.type}`;
};

function Car(color) {
  Vehicle.call(this, "car");
  this.color = color;
}
Car.prototype = Object.create(Vehicle.prototype);
Car.prototype.constructor = Car;
Car.prototype.honk = function () {
  return "beep!";
};

const myCar = new Car("red");
```

Answer:

1. What is `Object.getPrototypeOf(myCar)`?
2. What is `Object.getPrototypeOf(Object.getPrototypeOf(myCar))`?
3. What does `myCar.describe()` return?
4. Is `myCar instanceof Vehicle` true?

<details>
<summary>Answer</summary>

```
1. Object.getPrototypeOf(myCar) === Car.prototype  ✓
2. Object.getPrototypeOf(Car.prototype) === Vehicle.prototype  ✓
3. myCar.describe()
   → not on myCar → not on Car.prototype → found on Vehicle.prototype
   → returns 'I am a car'  ✓
4. myCar instanceof Vehicle
   → checks if Vehicle.prototype is anywhere in myCar's chain
   → chain: myCar → Car.prototype → Vehicle.prototype ← found!
   → true  ✓
```

</details>

---

### Exercise 2 — Implement `new` from scratch

```javascript
function myNew(Constructor, ...args) {
  // Implement the 4 steps of new
}

// Test:
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function () {
  return `Hi, I'm ${this.name}`;
};

const p = myNew(Person, "Alice");
console.log(p.greet()); // "Hi, I'm Alice"
console.log(p instanceof Person); // true
```

<details>
<summary>Solution</summary>

```javascript
function myNew(Constructor, ...args) {
  // Step 1: create object with Constructor.prototype as [[Prototype]]
  const obj = Object.create(Constructor.prototype);

  // Steps 2 & 3: call constructor with obj as `this`
  const returnValue = Constructor.apply(obj, args);

  // Step 4: return returnValue if it's an object, else return obj
  return returnValue !== null && typeof returnValue === "object"
    ? returnValue
    : obj;
}
```

</details>

---

### Exercise 3 — Fix the shared mutable state bug

```javascript
// ❌ This has a shared mutable state bug — find and fix it
function ShoppingCart(owner) {
  this.owner = owner;
}

ShoppingCart.prototype.items = [];
ShoppingCart.prototype.discount = 0;

ShoppingCart.prototype.addItem = function (item) {
  this.items.push(item);
};

ShoppingCart.prototype.setDiscount = function (pct) {
  this.discount = pct;
};

const cartA = new ShoppingCart("Alice");
const cartB = new ShoppingCart("Bob");

cartA.addItem("Book");
cartB.addItem("Phone");

console.log(cartA.items); // ['Book', 'Phone'] ← BUG!
```

<details>
<summary>Solution</summary>

```javascript
// Fix: initialize mutable state in constructor as own properties
function ShoppingCart(owner) {
  this.owner = owner;
  this.items = []; // own property — not shared
  this.discount = 0; // own property — not shared
}

// Methods on prototype (shared is fine for functions)
ShoppingCart.prototype.addItem = function (item) {
  this.items.push(item);
};

ShoppingCart.prototype.setDiscount = function (pct) {
  this.discount = pct;
};

const cartA = new ShoppingCart("Alice");
const cartB = new ShoppingCart("Bob");

cartA.addItem("Book");
cartB.addItem("Phone");

console.log(cartA.items); // ['Book'] ✓
console.log(cartB.items); // ['Phone'] ✓
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/01-execution-context.md`](./01-execution-context.md) — `this` binding in constructors
- [`javascript-core/05-closures.md`](./05-closures.md) — Closures vs prototype-based private state
- [`javascript-core/07-scope-chain.md`](./07-scope-chain.md) — Scope chain vs prototype chain
- [`patterns/05-proxy-pattern.md`](../patterns/05-proxy-pattern.md) — Proxy wrapping prototypes

---

<div align="center">

**Next:** [`javascript-core/07-scope-chain.md`](./07-scope-chain.md) →

</div>
