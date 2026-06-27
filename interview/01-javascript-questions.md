# 01 — JavaScript Interview Questions

> **"A JavaScript interview question isn't really asking you to recite a spec. It's asking whether you understand the runtime well enough to reason about what code will do before running it, and to explain the 'why' behind non-obvious behavior. The answer matters less than the thinking."**

This document compiles the most important JavaScript interview questions for mid-to-senior frontend positions, organized by topic, with thorough explanations for each. The focus is on understanding the underlying concepts — closures, the event loop, prototypal inheritance, type coercion — that let you reason correctly about JavaScript behavior rather than memorizing edge cases.

---

## 📚 Table of Contents

1. [Closures and Scope](#1-closures-and-scope)
2. [The Event Loop](#2-the-event-loop)
3. [Prototypes and Inheritance](#3-prototypes-and-inheritance)
4. [this Binding](#4-this-binding)
5. [Promises and Async/Await](#5-promises-and-asyncawait)
6. [Type Coercion and Equality](#6-type-coercion-and-equality)
7. [Functional Concepts](#7-functional-concepts)
8. [ES6+ Features](#8-es6-features)
9. [Memory and Performance](#9-memory-and-performance)

---

## 1. Closures and Scope

### Q: What is a closure? Give a practical example.

**Answer:**

> A closure is a function that retains access to its lexical scope even when executed outside that scope. In JavaScript, every function creates a closure over the variables in its surrounding scope at the time the function is defined.

```javascript
// Practical example: private state via closure
function createCounter(initial = 0) {
  let count = initial; // private variable — not accessible from outside

  return {
    increment: () => ++count,
    decrement: () => --count,
    value: () => count,
    reset: () => {
      count = initial;
    },
  };
}

const counter = createCounter(10);
counter.increment(); // 11
counter.increment(); // 12
counter.value(); // 12
counter.reset();
counter.value(); // 10

// `count` is inaccessible: counter.count is undefined
// Only the returned methods can modify it — classic module pattern
```

```javascript
// Classic pitfall: closures in loops
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // logs: 3, 3, 3 (not 0, 1, 2!)
}
// Why: all three callbacks close over the SAME `i` variable
// When they run (after the loop), i is already 3

// Fix 1: let (block scope — new i per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // logs: 0, 1, 2
}

// Fix 2: IIFE to capture current value
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}
```

---

### Q: What is the difference between `var`, `let`, and `const`?

```javascript
// VAR: function-scoped, hoisted (initialized to undefined), re-declarable
function example() {
  console.log(x); // undefined (hoisted, not TDZ)
  var x = 5;
  if (true) {
    var x = 10; // same x — var is function-scoped, not block-scoped
  }
  console.log(x); // 10
}

// LET: block-scoped, hoisted but in TDZ (Temporal Dead Zone), not re-declarable
{
  // console.log(y); // ReferenceError: Cannot access 'y' before initialization (TDZ)
  let y = 5;
  {
    let y = 10; // different y — block-scoped
  }
  console.log(y); // 5
}

// CONST: block-scoped, must be initialized, BINDING is immutable (not the value)
const obj = { a: 1 };
obj.a = 2; // ✅ works — the object itself is mutable
// obj = {};       // ❌ TypeError: Assignment to constant variable
const arr = [1, 2];
arr.push(3); // ✅ works
```

---

## 2. The Event Loop

### Q: Explain the JavaScript event loop and the difference between micro-tasks and macro-tasks.

**Answer:**

> JavaScript is single-threaded. The event loop continuously checks: is the call stack empty? If so, take the next task from the task queue and push it onto the stack. Micro-tasks (Promise callbacks, queueMicrotask) have their own queue that is fully drained after every task, before the next macro-task runs.

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0); // macro-task

Promise.resolve().then(() => console.log("3")); // micro-task

queueMicrotask(() => console.log("4")); // micro-task

console.log("5");

// Output: 1, 5, 3, 4, 2
//
// Execution order:
// 1. Synchronous: "1", "5"
// 2. Micro-task queue drained: "3", "4" (all microtasks before any macro-task)
// 3. Macro-task queue: "2"
```

```javascript
// Nested microtasks: microtasks can schedule more microtasks
// (runs to completion before any macro-task)
Promise.resolve()
  .then(() => {
    console.log("A");
    Promise.resolve().then(() => console.log("B")); // schedules another microtask
    return "ignored";
  })
  .then(() => console.log("C"));

setTimeout(() => console.log("D"), 0);

// Output: A, B, C, D
// "B" runs before "C" because it was scheduled while the micro-task queue
// was already draining — new microtasks scheduled mid-drain run before the queue is "empty"
```

---

### Q: What is `requestAnimationFrame` and when does it run in the event loop?

```javascript
// rAF callbacks run before each repaint, after macro-tasks and microtasks
// but before the browser paints the next frame

setTimeout(() => console.log("macro-task"), 0);
Promise.resolve().then(() => console.log("micro-task"));
requestAnimationFrame(() => console.log("rAF"));

// Approximate order:
// micro-task (always before next macro-task/rAF)
// rAF (before paint)
// paint
// macro-task (next event loop iteration)
// (though rAF vs setTimeout ordering can vary by browser and timing)
```

---

## 3. Prototypes and Inheritance

### Q: How does prototypal inheritance work in JavaScript?

**Answer:**

> Every JavaScript object has an internal `[[Prototype]]` link to another object (or null). When you access a property on an object and it's not found directly on the object, JavaScript follows the prototype chain — looking at the prototype object, then the prototype's prototype — until it finds the property or reaches null.

```javascript
const animal = {
  breathe() {
    return "breathing";
  },
};

const dog = Object.create(animal); // dog's [[Prototype]] = animal
dog.bark = function () {
  return "woof";
};

dog.bark(); // "woof" — found directly on dog
dog.breathe(); // "breathing" — not on dog, found on animal (prototype)
dog.toString(); // found on Object.prototype (end of chain)

// The chain:
// dog → animal → Object.prototype → null

Object.getPrototypeOf(dog) === animal; // true
```

### Q: What is the difference between `__proto__`, `prototype`, and `Object.getPrototypeOf()`?

```javascript
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function () {
  return `Hi, ${this.name}`;
};

const alice = new Person("Alice");

// `prototype`: property on CONSTRUCTOR FUNCTIONS only
// The object that will become [[Prototype]] of instances
Person.prototype; // { greet: [Function], constructor: Person }

// `[[Prototype]]`: the actual internal prototype link of an instance
// Accessible via Object.getPrototypeOf() (standard) or .__proto__ (legacy)
Object.getPrototypeOf(alice) === Person.prototype; // true
alice.__proto__ === Person.prototype; // true (but deprecated)

// Summary:
// Person.prototype → the prototype object (only on functions)
// Object.getPrototypeOf(alice) → how to access the prototype link on an instance
// .__proto__ → same but deprecated, avoid in production code
```

---

## 4. this Binding

### Q: Explain the four rules of `this` binding in JavaScript.

```javascript
// RULE 1: Default binding (standalone function call)
// this = global (window) in non-strict, undefined in strict mode
function logThis() {
  console.log(this);
}
logThis(); // window (non-strict) or undefined (strict)

// RULE 2: Implicit binding (method call)
// this = the object to the left of the dot
const obj = {
  name: "Alice",
  greet() {
    return this.name;
  },
};
obj.greet(); // 'Alice' — this = obj

// RULE 3: Explicit binding (call, apply, bind)
function greet(greeting) {
  return `${greeting}, ${this.name}`;
}
greet.call({ name: "Bob" }, "Hello"); // "Hello, Bob"
greet.apply({ name: "Carol" }, ["Hi"]); // "Hi, Carol"
const boundGreet = greet.bind({ name: "Dave" });
boundGreet("Hey"); // "Hey, Dave"

// RULE 4: new binding
// this = the newly created object
function Person(name) {
  this.name = name;
  // implicitly: return this;
}
const alice = new Person("Alice");
alice.name; // 'Alice'

// ARROW FUNCTIONS: no own `this` — inherit from lexical scope
const timer = {
  seconds: 0,
  start() {
    setInterval(() => {
      this.seconds++; // `this` is the `timer` object (lexical)
    }, 1000);
  },
};
// If setInterval used a regular function: this would be window (or undefined in strict)
// Arrow function: `this` is whatever `this` was in start() = timer
```

---

## 5. Promises and Async/Await

### Q: What states can a Promise be in? What is Promise chaining?

```javascript
// Three states: pending, fulfilled, rejected
// Transitions are ONE-WAY: once settled (fulfilled/rejected), state cannot change

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => resolve("value"), 1000);
});
// p1: pending → (after 1s) → fulfilled with 'value'

const p2 = new Promise((resolve, reject) => {
  reject(new Error("something went wrong"));
});
// p2: pending → immediately → rejected

// PROMISE CHAINING: .then() returns a NEW Promise
// The returned value becomes the fulfillment value of the next .then()
fetch("/api/user")
  .then((response) => response.json()) // returns Promise<data>
  .then((data) => data.userId) // returns userId string
  .then((userId) => fetch(`/api/orders/${userId}`)) // returns Promise<Response>
  .then((response) => response.json())
  .catch((error) => handleError(error)); // catches any rejection in the chain
```

### Q: What is the difference between `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`?

```javascript
const p1 = fetch("/api/users");
const p2 = fetch("/api/products");
const p3 = Promise.reject(new Error("failed"));

// Promise.all: resolves when ALL resolve; rejects immediately if ANY rejects
await Promise.all([p1, p2]); // resolves with [response1, response2]
await Promise.all([p1, p2, p3]); // REJECTS immediately (p3 fails)

// Promise.allSettled: resolves when ALL settle (regardless of success/failure)
const results = await Promise.allSettled([p1, p2, p3]);
// [{ status: 'fulfilled', value: resp1 },
//  { status: 'fulfilled', value: resp2 },
//  { status: 'rejected',  reason: Error }]
// Never rejects — always gives you all results

// Promise.race: resolves/rejects as soon as the FIRST one settles
await Promise.race([p1, p2]); // whichever responds first

// Promise.any: resolves with FIRST fulfillment; rejects if ALL reject
await Promise.any([p1, p2, p3]); // resolves with first success (ignores p3 failure)
// Only rejects if ALL promises reject → AggregateError
```

### Q: How does `async/await` relate to Promises?

```javascript
// async/await is syntactic sugar over Promises

// These are equivalent:
function getData() {
  return fetch("/api/data")
    .then((r) => r.json())
    .then((data) => data.items)
    .catch((err) => {
      throw err;
    });
}

async function getData() {
  const response = await fetch("/api/data"); // awaits the Promise
  const data = await response.json();
  return data.items;
  // Errors thrown here automatically reject the returned Promise
}

// await pauses the async function's execution until the Promise resolves
// Does NOT block the main thread — other code runs while waiting

// Error handling:
async function withErrorHandling() {
  try {
    const data = await fetch("/api/data").then((r) => r.json());
    return data;
  } catch (err) {
    console.error("Failed:", err);
    throw err; // re-throw if needed
  }
}
```

---

## 6. Type Coercion and Equality

### Q: What is the difference between `==` and `===`?

```javascript
// === (strict): no type coercion — must be same type AND value
1 === 1; // true
1 === "1"; // false (different types)
null === undefined; // false

// == (loose): performs type coercion
1 == "1"; // true (string coerced to number)
0 == false; // true (false coerced to 0)
"" == false; // true (both coerce to 0)
null == undefined; // true (special case — only equal to each other)
null == 0; // false (null only == undefined with ==)

// Always use === unless you specifically need null/undefined comparison:
if (value == null) {
  // catches both null AND undefined
  // equivalent to: if (value === null || value === undefined)
}
```

### Q: What are falsy values in JavaScript?

```javascript
// FALSY VALUES (8 total):
false
0          // also: -0, 0n (BigInt zero)
''         // empty string (also: "", ``)
null
undefined
NaN
document.all // (legacy, unusual)

// Everything else is truthy:
'0'        // truthy! (non-empty string)
[]         // truthy! (empty array)
{}         // truthy! (empty object)
-1         // truthy! (non-zero number)

// Common gotcha:
if ([]) console.log('array is truthy');   // logs!
if ({}) console.log('object is truthy');  // logs!
if (0) console.log('not logged');         // not logged
```

---

## 7. Functional Concepts

### Q: What are pure functions? What is referential transparency?

```javascript
// PURE FUNCTION: same inputs → same output, no side effects
function add(a, b) {
  return a + b; // pure: no mutation, no external state
}

// IMPURE: reads/writes external state or has side effects
let total = 0;
function addToTotal(n) {
  total += n; // impure: modifies external variable
  return total;
}

// REFERENTIAL TRANSPARENCY: you can replace a function call with its result
// without changing program behavior
const x = add(2, 3); // 5 — referentially transparent
// const x = 5;        // equivalent substitution

// Benefits of pure functions:
// - Testable: no mocking external state
// - Memoizable: same input always gives same output (safe to cache)
// - Composable: predictable behavior when combined
```

### Q: What is currying? Give an example.

```javascript
// CURRYING: transform a function that takes multiple arguments into
// a sequence of functions each taking one argument

// Normal function:
function multiply(a, b) {
  return a * b;
}

// Curried version:
function multiply(a) {
  return function (b) {
    return a * b;
  };
}

const double = multiply(2); // partially applied — a = 2
double(5); // 10
double(7); // 14

// Arrow function version:
const multiply = (a) => (b) => a * b;
const triple = multiply(3);
triple(4); // 12

// Practical use: create specialized functions from generic ones
const add = (a) => (b) => a + b;
const add10 = add(10);
[1, 2, 3].map(add10); // [11, 12, 13]
```

---

## 8. ES6+ Features

### Q: What are generators? What is a use case for them?

```javascript
// Generators: functions that can be paused and resumed
// yield: pause and return a value
// next(): resume until the next yield

function* countdown(from) {
  while (from > 0) {
    yield from--; // pause here, return `from`, then resume
  }
}

const gen = countdown(3);
gen.next(); // { value: 3, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: 1, done: false }
gen.next(); // { value: undefined, done: true }

// Iterable: can be used with for...of
for (const n of countdown(5)) {
  console.log(n); // 5, 4, 3, 2, 1
}

// USE CASE: infinite sequences without consuming memory
function* naturals() {
  let n = 1;
  while (true) yield n++;
}
const nums = naturals();
nums.next().value; // 1
nums.next().value; // 2
// only computes values on demand
```

### Q: What is destructuring? What is the rest/spread operator?

```javascript
// DESTRUCTURING: extract values into variables
const {
  name,
  age,
  address: { city },
} = user; // nested
const [first, second, ...rest] = array; // array

// Default values:
const { theme = "light", language = "en" } = settings;

// Rename while destructuring:
const { firstName: first_name } = user;

// REST (...): collect remaining elements into an array/object
function sum(...numbers) {
  // rest parameter
  return numbers.reduce((a, b) => a + b, 0);
}
const { id, ...rest } = obj; // object rest: rest has everything except id
const [head, ...tail] = arr; // array rest

// SPREAD (...): expand an iterable or object into individual elements
const combined = [...arr1, ...arr2]; // array spread
const merged = { ...defaults, ...overrides }; // object spread (later wins)
const copy = [...original]; // shallow copy

// Function call spread:
Math.max(...[1, 2, 3]); // equivalent to Math.max(1, 2, 3)
```

---

## 9. Memory and Performance

### Q: What is the difference between shallow and deep copy?

```javascript
// SHALLOW COPY: copies the object, but nested objects are still shared references
const original = { a: 1, b: { c: 2 } };

const shallow1 = { ...original }; // spread
const shallow2 = Object.assign({}, original);
const shallow3 = Array.from(arr); // for arrays
const shallow4 = [...arr]; // for arrays

shallow1.a = 99; // doesn't affect original (primitive)
shallow1.b.c = 99; // DOES affect original! (shared reference to b)
original.b.c; // 99 — mutated!

// DEEP COPY: fully independent copy, no shared references
// Option 1: structuredClone (modern, handles most types)
const deep = structuredClone(original); // handles cycles, Date, Map, Set, etc.
deep.b.c = 99; // doesn't affect original.b.c

// Option 2: JSON (limited — loses functions, undefined, Date becomes string)
const deep2 = JSON.parse(JSON.stringify(original));
// Don't use for: functions, undefined, Date, Symbol, BigInt, circular refs

// Option 3: Lodash _.cloneDeep (handles edge cases well)
import { cloneDeep } from "lodash";
const deep3 = cloneDeep(original);
```

### Q: How does garbage collection work in JavaScript?

```javascript
// MARK-AND-SWEEP algorithm:
// 1. Start from "roots" (global variables, call stack)
// 2. Mark all objects reachable from roots
// 3. Sweep (collect) all unmarked objects

let obj = { data: "important" }; // obj is reachable → not collected
obj = null; // original object is no longer reachable → eligible for GC

// REFERENCE COUNTING (older, V8 uses mark-and-sweep now):
// Count references to each object
// When count reaches 0: collect
// Problem: circular references never reach 0 (even when unreachable from roots)

// Circular reference (not a problem for mark-and-sweep):
let a = {};
let b = {};
a.ref = b;
b.ref = a;
a = null;
b = null;
// Both are now unreachable from any root → GC can collect them
// (mark-and-sweep handles this; reference counting would leak)

// MEMORY LEAK: an object that is reachable (some reference exists) but no longer needed
// Common sources: event listeners not removed, closures holding large objects,
// global variables accumulating data, WeakMap vs Map for object metadata
```

---

## Quick-Reference: Common Output Questions

```javascript
// Q: What does this log?
typeof null;         // "object" (historical bug in JS, cannot be changed)
typeof undefined;    // "undefined"
typeof NaN;          // "number" (NaN is "Not a Number" but type is number)
typeof function(){}; // "function"

NaN === NaN;         // false (NaN is the only value not equal to itself)
Number.isNaN(NaN);   // true (the correct way to check)

0.1 + 0.2 === 0.3;   // false (floating point imprecision: 0.30000000000000004)
Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON; // true (correct comparison)

[] + []  // "" (both coerced to empty string, concatenated)
[] + {}  // "[object Object]" ([] → "", {} → "[object Object]")
{} + []  // 0 (in some contexts: {} is a block, +[] is unary plus on [])

// Tricky equality:
[] == ![]  // true (![] is false, [] == false, [] coerces to 0, false is 0)
```
