# 24 — ES6+ Modern Syntax

> **"Modern JavaScript syntax isn't just shorter — it's more expressive. Destructuring communicates 'I only need these parts.' Optional chaining says 'this might not exist.' `for...of` says 'give me the values.' Each feature is a compact vocabulary for a recurring intent. Using them fluently makes your code read like your thinking."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [let and const (Block Scope)](#1-let-and-const-block-scope)
2. [Arrow Functions Recap](#2-arrow-functions-recap)
3. [Destructuring Deep Dive](#3-destructuring-deep-dive)
4. [Spread and Rest Everywhere](#4-spread-and-rest-everywhere)
5. [Symbols](#5-symbols)
6. [Iterables and for…of](#6-iterables-and-forof)
7. [Map and Set](#7-map-and-set)
8. [WeakMap and WeakSet](#8-weakmap-and-weakset)
9. [Classes](#9-classes)
10. [Optional Chaining and Nullish Coalescing (Recap)](#10-optional-chaining-and-nullish-coalescing-recap)
11. [Logical Assignment Operators](#11-logical-assignment-operators)
12. [Object.hasOwn and at()](#12-objecthasown-and-at)
13. [Exercises](#13-exercises)

---

## 1. let and const (Block Scope)

```javascript
// Block scoping: variables live only within their { } block
{
  let x = 10;
  const y = 20;
}
// x and y are not accessible here

// TDZ (Temporal Dead Zone): accessing let/const before declaration throws
console.log(a); // ❌ ReferenceError — TDZ
let a = 5;

// const: the BINDING is immutable, not the value
const arr = [1, 2, 3];
arr.push(4); // ✅ OK — mutating the array
arr = [5]; // ❌ TypeError — reassigning the binding

// Practical implication: prefer const everywhere, use let only when reassignment is needed
// A `const` declaration signals "this variable won't point to a different value"
```

---

## 2. Arrow Functions Recap

```javascript
// Concise syntax
const double = (x) => x * 2;
const add = (a, b) => a + b;
const noop = () => {};

// No own `this` — lexically inherits it
class Btn {
  constructor() {
    this.clicks = 0;
  }
  onClick() {
    document.addEventListener("click", () => this.clicks++); // ✅ `this` = Btn instance
  }
}

// Not suitable as methods (no own `this`)
const obj = {
  name: "Alice",
  greet: () => `Hi, ${this.name}`, // ❌ `this` is the outer scope, not obj
  greetOk() {
    return `Hi, ${this.name}`;
  }, // ✅ regular method syntax
};
```

---

## 3. Destructuring Deep Dive

```javascript
// Array destructuring with computed/complex patterns
const matrix = [
  [1, 2],
  [3, 4],
];
const [[a, b], [c, d]] = matrix;

// Swap in place
let x = 1,
  y = 2;
[x, y] = [y, x];

// Ignore elements
const [, , third] = [1, 2, 3]; // third = 3

// Object destructuring: rename + default + nested + rest
const { name: n = "Guest", address: { city = "Unknown" } = {}, ...rest } = user;

// In function parameters with complex defaults
function connect({
  host = "localhost",
  port = 5432,
  ssl = false,
  credentials: { user = "root", password = "" } = {},
} = {}) {
  return { host, port, ssl, user, password };
}
connect(); // all defaults
connect({ port: 3306, ssl: true }); // partial override

// Mixed array + object destructuring
const [{ name: firstName }, { name: lastName }] = [
  { name: "Alice" },
  { name: "Smith" },
];

// Destructuring iterables (anything with Symbol.iterator)
const [first, ...rest] = new Set([1, 2, 3, 4]); // Set is iterable
const [key, value] = new Map([["a", 1]]).entries().next().value;
```

---

## 4. Spread and Rest Everywhere

```javascript
// In function calls
const args = [1, 2, 3];
Math.max(...args);

// In array literals
const merged = [...arr1, 0, ...arr2];

// In object literals (shallow merge)
const config = { ...defaults, ...overrides };

// Clone then modify pattern
const updatedUser = { ...user, age: user.age + 1 };

// Remove keys via rest
const { password, __v, ...safeUser } = dbRecord;

// Collect remaining function arguments
function log(level, ...messages) {
  messages.forEach((m) => console.log(`[${level}]`, m));
}

// Spread a string into array of characters
[..."hello"]; // ['h', 'e', 'l', 'l', 'o']

// Spread a Set to remove duplicates
[...new Set([1, 2, 2, 3, 3])]; // [1, 2, 3]
```

---

## 5. Symbols

```javascript
// Symbol: globally unique primitive, often used as unique object keys
const ID = Symbol("id");
const NAME = Symbol("name");

const user = {
  [ID]: 123,
  [NAME]: "Alice",
  email: "alice@example.com",
};

user[ID]; // 123
Object.keys(user); // ['email'] — Symbol keys excluded
JSON.stringify(user); // '{"email":"alice@example.com"}' — symbols excluded
Object.getOwnPropertySymbols(user); // [Symbol(id), Symbol(name)]

// Well-known Symbols (protocol hooks into built-in behavior)
class MyArray {
  [Symbol.iterator]() {
    /* custom iteration */
  }
}
class MyClass {
  get [Symbol.toStringTag]() {
    return "MyClass";
  } // Object.prototype.toString output
}
Object.prototype.toString.call(new MyClass()); // '[object MyClass]'

// Symbol.for: global symbol registry (shared across files/modules)
const A = Symbol.for("shared-key");
const B = Symbol.for("shared-key");
A === B; // true — same global symbol
```

---

## 6. Iterables and for…of

```javascript
// An iterable is any object with a [Symbol.iterator] method
// Built-in iterables: Array, String, Map, Set, arguments, generators, NodeList

for (const item of [1, 2, 3]) {
  /* arrays */
}
for (const char of "hello") {
  /* strings */
}
for (const [k, v] of new Map()) {
  /* Map entries */
}
for (const item of new Set([1, 2, 3])) {
  /* Set values */
}

// entries(), keys(), values() return iterables
for (const [i, val] of ["a", "b", "c"].entries()) {
  /* i=0,1,2 */
}
for (const key of Object.keys(obj)) {
  /* object keys */
}

// Making a custom class iterable
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    return {
      next() {
        return current <= end
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      },
    };
  }
}

for (const n of new Range(1, 5)) {
  console.log(n); // 1, 2, 3, 4, 5
}

[...new Range(1, 3)]; // [1, 2, 3] — spread works too (uses the iterator)
```

---

## 7. Map and Set

```javascript
// MAP: key-value pairs where keys can be ANY type (not just strings)
const map = new Map();
map.set("name", "Alice");
map.set(42, "the answer");
map.set({ id: 1 }, "object key"); // object as key

map.get("name"); // 'Alice'
map.has("name"); // true
map.size; // 3
map.delete("name");
map.clear(); // removes all entries

// Iteration
for (const [key, value] of map) {
  /* ... */
}
map.keys(); // iterable of keys
map.values(); // iterable of values
map.entries(); // iterable of [key, value] pairs

// Initialize from entries
const map2 = new Map([
  ["a", 1],
  ["b", 2],
  ["c", 3],
]);

// Map vs Object: when to use which
// Map: keys can be non-strings, maintain insertion order, frequent add/delete
// Object: static known keys, JSON serialization, prototype methods

// SET: unique values, any type
const set = new Set([1, 2, 2, 3, 3, 3]);
set.size; // 3
set.has(2); // true
set.add(4);
set.delete(1);

// Remove duplicates from array
const unique = [...new Set(array)];

// Set operations
const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5, 6]);
const union = new Set([...a, ...b]); // {1,2,3,4,5,6}
const intersection = new Set([...a].filter((x) => b.has(x))); // {3,4}
const difference = new Set([...a].filter((x) => !b.has(x))); // {1,2}
```

---

## 8. WeakMap and WeakSet

```javascript
// WeakMap: keys must be OBJECTS, held by weak reference
// When the key object is garbage-collected, the entry is automatically removed
const cache = new WeakMap();

function processUser(user) {
  if (cache.has(user)) return cache.get(user);
  const result = expensiveComputation(user);
  cache.set(user, result); // no memory leak if `user` is later GC'd
  return result;
}

// Only 4 operations: get, set, has, delete
// No iteration (size, keys, values, forEach are NOT available)
// Use case: private data for objects, DOM element metadata

// WeakSet: a set of objects (no primitives), weakly held
const rendered = new WeakSet();

function renderOnce(element) {
  if (rendered.has(element)) return;
  render(element);
  rendered.add(element); // auto-removed when element is GC'd
}

// WeakRef: weak reference to a value (GC may collect it)
const weakRef = new WeakRef(largeObject);
const obj = weakRef.deref(); // undefined if GC'd, or the object if still alive
if (obj) {
  use(obj);
}
```

---

## 9. Classes

```javascript
class Animal {
  // Class field (ES2022)
  #name; // private field — not accessible outside the class
  type = "animal"; // public field with default

  constructor(name) {
    this.#name = name;
  }

  // Instance method
  speak() {
    return `${this.#name} makes a sound`;
  }

  // Getter / setter
  get name() {
    return this.#name;
  }
  set name(v) {
    if (typeof v !== "string") throw new TypeError("Name must be a string");
    this.#name = v;
  }

  // Static method (on the class itself, not instances)
  static create(name) {
    return new Animal(name);
  }

  // Static field
  static count = 0;
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // must call super() before using this
    this.breed = breed;
    Animal.count++;
  }

  speak() {
    return `${super.speak()} — Woof!`; // super.method() calls parent
  }

  // Private method
  #validate() {
    /* ... */
  }
}

const dog = new Dog("Rex", "Labrador");
dog.speak(); // 'Rex makes a sound — Woof!'
dog.name; // 'Rex' (getter)
dog.name = "Max"; // setter
// dog.#name; // ❌ SyntaxError — private fields are truly private

// instanceof and class identity
dog instanceof Dog; // true
dog instanceof Animal; // true (prototype chain)
```

---

## 10. Optional Chaining and Nullish Coalescing (Recap)

```javascript
// Optional chaining (?.) — short-circuits to undefined if LHS is null/undefined
const city = user?.address?.city;
const first = arr?.[0];
const val = obj?.method?.();

// Nullish coalescing (??) — fallback ONLY for null/undefined
const name = user?.name ?? "Anonymous"; // won't treat empty string as missing
const port = config.port ?? 3000; // 0 is a valid port, ?? preserves it

// Nullish assignment (??=)
config.timeout ??= 5000; // only sets if currently null/undefined
config.debug ||= false; // sets if currently falsy
config.retries &&= config.retries + 1; // only updates if currently truthy
```

---

## 11. Logical Assignment Operators

```javascript
// Introduced in ES2021
// a ??= b  ←→  a = a ?? b  (assign if null/undefined)
// a ||= b  ←→  a = a || b  (assign if falsy)
// a &&= b  ←→  a = a && b  (assign if truthy)

// Common pattern: initialize object properties with defaults
function init(options = {}) {
  options.timeout ??= 3000; // set only if not already set
  options.retries ??= 3;
  options.debug ||= false; // set if falsy (including undefined)
  options.enabled &&= options.enabled && validate(options); // update if set
  return options;
}
```

---

## 12. Object.hasOwn and at()

```javascript
// Object.hasOwn() — ES2022, safer than hasOwnProperty
// Works correctly even when the object has no prototype
const obj = Object.create(null); // no Object.prototype methods
obj.x = 1;
Object.hasOwn(obj, "x"); // true ✅
// obj.hasOwnProperty('x') // ❌ TypeError — hasOwnProperty doesn't exist

// at() — ES2022: index from the end with negative numbers
const arr = [1, 2, 3, 4, 5];
arr.at(-1); // 5 (last)
arr.at(-2); // 4 (second to last)
"hello".at(-1); // 'o'

// Array grouping: Object.groupBy and Map.groupBy (ES2024)
const people = [
  { name: "Alice", dept: "Eng" },
  { name: "Bob", dept: "Marketing" },
  { name: "Carol", dept: "Eng" },
];
const byDept = Object.groupBy(people, (p) => p.dept);
// { Eng: [{name:'Alice',...},{name:'Carol',...}], Marketing: [{name:'Bob',...}] }
```

---

## 13. Exercises

### Exercise 1 — Transform with modern syntax

```javascript
// Using only modern JS features (destructuring, spread, optional chaining, etc.),
// write a function that takes an array of user objects and returns
// an array of display strings, sorted alphabetically, skipping users without a name.
//
// Input:  [{ name: 'Charlie', age: 30 }, { age: 25 }, { name: 'Alice', age: 20 }]
// Output: ['Alice (20)', 'Charlie (30)']
```

<details>
<summary>Solution</summary>

```javascript
function getUserDisplays(users) {
  return users
    .filter((u) => u?.name)
    .sort((a, b) => a.name.localeCompare(b.name))
    .map(({ name, age }) => `${name} (${age})`);
}
```

</details>

---

## 🔗 Related Topics

- [`16-variables-and-data-types.md`](./16-variables-and-data-types.md) — var/let/const foundations
- [`25-modules-and-bundling.md`](./25-modules-and-bundling.md) — ES modules syntax
- [`26-iterators-and-generators.md`](./26-iterators-and-generators.md) — Symbol.iterator in depth
