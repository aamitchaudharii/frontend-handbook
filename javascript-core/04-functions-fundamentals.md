# 19 — Functions — Fundamentals

> **"A function is the single most important abstraction in JavaScript. Every technique — callbacks, promises, hooks, closures, higher-order functions — is built on the premise that functions are values, and values can be stored, passed, and returned just like any other data."**

🟢 **Level: Beginner → Intermediate**

---

## 📚 Table of Contents

1. [Function Declarations vs Expressions](#1-function-declarations-vs-expressions)
2. [Arrow Functions](#2-arrow-functions)
3. [Parameters and Arguments](#3-parameters-and-arguments)
4. [Default Parameters](#4-default-parameters)
5. [Rest Parameters and Spread](#5-rest-parameters-and-spread)
6. [Functions as Values (First-Class Functions)](#6-functions-as-values-first-class-functions)
7. [Higher-Order Functions](#7-higher-order-functions)
8. [IIFE — Immediately Invoked Function Expressions](#8-iife--immediately-invoked-function-expressions)
9. [Pure Functions](#9-pure-functions)
10. [Recursion](#10-recursion)
11. [Common Mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)

---

## 1. Function Declarations vs Expressions

```javascript
// DECLARATION: hoisted — can be called before its definition in the code
function greet(name) {
  return `Hello, ${name}!`;
}
greet("Alice"); // "Hello, Alice!"

// Hoisting in action:
sayHi(); // ✅ works — declaration is hoisted to the top of the scope
function sayHi() {
  console.log("Hi!");
}

// EXPRESSION: NOT hoisted — must be defined before calling
const greet = function (name) {
  return `Hello, ${name}!`;
};
// greet('Alice') before this line → TypeError: greet is not a function

// Named function expression (name is local to the function body — useful for recursion)
const factorial = function fact(n) {
  return n <= 1 ? 1 : n * fact(n - 1); // can call `fact` inside
};
// fact is not accessible here — only `factorial` is

// Which to use?
// Declaration: for top-level utility functions that need to be callable anywhere in the scope
// Expression: for functions assigned to variables, passed as arguments, or returned
```

---

## 2. Arrow Functions

```javascript
// Concise syntax for function expressions
const double = (x) => x * 2; // single expression: implicit return
const add = (a, b) => a + b; // two parameters
const square = (x) => x * x; // one param: parens optional
const getObj = () => ({ key: "val" }); // returning an object: wrap in ()

// Block body: explicit return required
const clamp = (value, min, max) => {
  if (value < min) return min;
  if (value > max) return max;
  return value;
};

// KEY DIFFERENCE from regular functions: arrow functions do NOT have their own `this`
// They capture `this` from the ENCLOSING lexical scope

class Timer {
  constructor() {
    this.seconds = 0;
  }

  start() {
    // ✅ Arrow function: `this` refers to the Timer instance (lexical scope)
    setInterval(() => {
      this.seconds++; // `this` is the Timer instance
    }, 1000);

    // ❌ Regular function: `this` is undefined (strict mode) or window (sloppy)
    // setInterval(function() {
    //   this.seconds++; // `this` is NOT the Timer instance!
    // }, 1000);
  }
}

// Other arrow function limitations:
// - Cannot be used as constructors: new (() => {}) → TypeError
// - No `arguments` object (use rest params instead)
// - No `prototype` property
// - Cannot use `yield` (can't be generators)
```

---

## 3. Parameters and Arguments

```javascript
function describe(name, age, city) {
  return `${name}, ${age}, from ${city}`;
}

describe("Alice", 30, "NY"); // "Alice, 30, from NY"
describe("Bob", 25); // "Bob, 25, from undefined" — missing args = undefined
describe("Carol", 28, "LA", "extra"); // extra args ignored

// arguments object (available in regular functions, not arrow functions)
function sum() {
  let total = 0;
  for (let i = 0; i < arguments.length; i++) {
    total += arguments[i];
  }
  return total;
}
sum(1, 2, 3, 4); // 10

// ✅ Modern: use rest parameters instead of `arguments`
function sumRest(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}
sumRest(1, 2, 3, 4); // 10
```

---

## 4. Default Parameters

```javascript
// Default values for parameters when undefined is passed (or parameter is omitted)
function greet(name = "World", greeting = "Hello") {
  return `${greeting}, ${name}!`;
}
greet(); // "Hello, World!"
greet("Alice"); // "Hello, Alice!"
greet("Alice", "Hi"); // "Hi, Alice!"
greet(undefined, "Hey"); // "Hey, World!" — undefined triggers the default
greet(null, "Hey"); // "Hey, null!"   — null does NOT trigger the default

// Defaults can reference earlier parameters
function createBox(width, height = width, depth = width) {
  return { width, height, depth };
}
createBox(5); // { width: 5, height: 5, depth: 5 }
createBox(5, 10); // { width: 5, height: 10, depth: 5 }

// Defaults can be expressions (evaluated at call time, not definition time)
function getTimestamp(date = new Date()) {
  return date.toISOString();
}
```

---

## 5. Rest Parameters and Spread

```javascript
// REST: collect remaining arguments into an array
function log(level, ...messages) {
  console.log(`[${level}]`, messages.join(", "));
}
log("INFO", "User logged in", "Session started"); // [INFO] User logged in, Session started

// Rest must be the LAST parameter
// function bad(a, ...rest, b) {} // SyntaxError

// SPREAD: expand an array/iterable into individual arguments
const numbers = [1, 2, 3, 4, 5];
Math.max(...numbers); // 5 — same as Math.max(1, 2, 3, 4, 5)
Math.min(...numbers); // 1

// Spread in arrays
const first = [1, 2, 3];
const second = [4, 5, 6];
const combined = [...first, ...second]; // [1, 2, 3, 4, 5, 6]
const copy = [...first]; // shallow copy

// Spread in objects
const defaults = { theme: "light", lang: "en" };
const user = { name: "Alice", lang: "fr" };
const merged = { ...defaults, ...user }; // { theme: 'light', lang: 'fr', name: 'Alice' }
// later values override earlier ones

// Practical: passing an array as arguments
function sum(a, b, c) {
  return a + b + c;
}
const args = [1, 2, 3];
sum(...args); // 6
```

---

## 6. Functions as Values (First-Class Functions)

In JavaScript, functions are **first-class citizens** — they can be stored, passed, and returned just like any other value.

```javascript
// Stored in a variable (already seen above)
const greet = function (name) {
  return `Hi, ${name}`;
};

// Stored in an object (a method)
const calculator = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
};
calculator.add(3, 4); // 7

// Stored in an array
const operations = [(x) => x + 1, (x) => x * 2, (x) => x ** 2];
operations[1](5); // 10

// Passed as an argument (callback)
[1, 2, 3].forEach(function (item) {
  console.log(item);
});

// Returned from a function
function multiplier(factor) {
  return function (number) {
    // returns a NEW function
    return number * factor;
  };
}
const double = multiplier(2);
const triple = multiplier(3);
double(5); // 10
triple(5); // 15
```

---

## 7. Higher-Order Functions

A **higher-order function** is a function that accepts a function as an argument OR returns a function.

```javascript
// Accepting a function as argument
function applyTwice(fn, value) {
  return fn(fn(value));
}
applyTwice((x) => x + 1, 5); // 7  (5 → 6 → 7)
applyTwice((x) => x * 2, 3); // 12 (3 → 6 → 12)

// Returning a function
function makeAdder(x) {
  return (y) => x + y;
}
const add5 = makeAdder(5);
add5(3); // 8
add5(10); // 15

// Built-in higher-order array methods
const nums = [1, 2, 3, 4, 5];

nums.map((n) => n * 2); // [2, 4, 6, 8, 10]
nums.filter((n) => n % 2 === 0); // [2, 4]
nums.reduce((acc, n) => acc + n, 0); // 15
nums.some((n) => n > 4); // true
nums.every((n) => n > 0); // true
nums.find((n) => n > 3); // 4
nums.findIndex((n) => n > 3); // 3

// Function composition via higher-order functions
const compose =
  (...fns) =>
  (x) =>
    fns.reduceRight((v, f) => f(v), x);
const pipe =
  (...fns) =>
  (x) =>
    fns.reduce((v, f) => f(v), x);

const process = pipe(
  (x) => x + 1, // 5 + 1 = 6
  (x) => x * 2, // 6 * 2 = 12
  (x) => x - 3, // 12 - 3 = 9
);
process(5); // 9
```

---

## 8. IIFE — Immediately Invoked Function Expressions

```javascript
// Syntax: (function definition)(arguments)
(function () {
  console.log("Runs immediately!");
})();

// Arrow function version
(() => {
  console.log("Also runs immediately!");
})();

// WHY use IIFEs?
// 1. Create a private scope (before ES modules / block scope existed)
(function () {
  const privateVar = "cannot be accessed outside";
  // all code here has its own scope
})();
// privateVar is not accessible here

// 2. Avoid polluting the global scope in older code (still seen in legacy)
// 3. Initialize something immediately with a complex expression
const config = (() => {
  const env = process.env.NODE_ENV;
  return {
    isProduction: env === "production",
    apiUrl:
      env === "production"
        ? "https://api.example.com"
        : "http://localhost:3000",
  };
})();
```

---

## 9. Pure Functions

```javascript
// PURE: same input → same output, no side effects
function add(a, b) {
  return a + b; // only depends on arguments, affects nothing outside
}

// IMPURE: depends on or modifies external state
let total = 0;
function addToTotal(n) {
  total += n; // modifies external state
  return total; // result depends on external state
}

// Another impure function: same input, different output
function getRandomNumber() {
  return Math.random(); // output changes each call
}

// Why pure functions matter:
// ✅ Easy to test (no mocking needed)
// ✅ Predictable (same result every time)
// ✅ Composable (safe to combine)
// ✅ Memoizable (safe to cache results)

// Making impure functions pure
// ❌ Impure: modifies the input array
function appendItem(arr, item) {
  arr.push(item); // mutates the original
  return arr;
}

// ✅ Pure: creates a new array instead
function appendItem(arr, item) {
  return [...arr, item]; // original unchanged
}
```

---

## 10. Recursion

```javascript
// A function that calls itself — must have a BASE CASE to stop
function factorial(n) {
  if (n <= 1) return 1; // base case — stops recursion
  return n * factorial(n - 1); // recursive case
}
factorial(5); // 120 (5 × 4 × 3 × 2 × 1)

// Fibonacci (classic recursion example)
function fibonacci(n) {
  if (n <= 1) return n; // base cases: fib(0) = 0, fib(1) = 1
  return fibonacci(n - 1) + fibonacci(n - 2);
}
// ⚠️ Exponential time — memoize for performance:
function fibMemo(n, memo = {}) {
  if (n in memo) return memo[n];
  if (n <= 1) return n;
  memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  return memo[n];
}

// Recursion on nested data structures
function flattenDeep(arr) {
  return arr.reduce(
    (acc, item) =>
      Array.isArray(item)
        ? [...acc, ...flattenDeep(item)] // recurse into nested arrays
        : [...acc, item],
    [],
  );
}
flattenDeep([1, [2, [3, [4]]]]); // [1, 2, 3, 4]

// ⚠️ Stack overflow risk: every recursive call adds a frame to the call stack
// Max call stack size is ~10,000-15,000 in most JS engines
// For large datasets, prefer iteration over recursion
```

---

## 11. Common Mistakes

### Mistake 1 — Calling a function vs referencing it

```javascript
// ❌ Calls the function immediately and passes its RETURN VALUE as the callback
setTimeout(greet(), 1000); // greet() runs now, undefined passed to setTimeout

// ✅ Passes the function REFERENCE — setTimeout calls it later
setTimeout(greet, 1000);
setTimeout(() => greet("Alice"), 1000); // with arguments: wrap in arrow function
```

### Mistake 2 — `this` in regular function callback

```javascript
class Counter {
  constructor() {
    this.count = 0;
  }
  start() {
    // ❌ `this` inside the callback is NOT the Counter instance
    setInterval(function () {
      this.count++;
    }, 1000); // `this` is undefined or window

    // ✅ Option 1: arrow function preserves `this`
    setInterval(() => {
      this.count++;
    }, 1000);

    // ✅ Option 2: bind
    setInterval(
      function () {
        this.count++;
      }.bind(this),
      1000,
    );
  }
}
```

### Mistake 3 — Forgetting return in a function

```javascript
// ❌ No return — function returns undefined
function double(n) {
  n * 2; // computed but not returned!
}
double(5); // undefined

// ✅
function double(n) {
  return n * 2;
}
```

---

## 12. Exercises

### Exercise 1 — Implement `once`

```javascript
// Implement a function that ensures fn is only called once,
// returning the same value on subsequent calls.
function once(fn) {
  /* ... */
}

const init = once(() => {
  console.log("Initialized");
  return 42;
});
init(); // logs "Initialized", returns 42
init(); // returns 42, no log
init(); // returns 42, no log
```

<details>
<summary>Solution</summary>

```javascript
function once(fn) {
  let called = false;
  let result;
  return function (...args) {
    if (!called) {
      result = fn.apply(this, args);
      called = true;
    }
    return result;
  };
}
```

</details>

---

## 🔗 Related Topics

- [`05-closures.md`](./05-closures.md) — Functions capturing outer variables
- [`07-scope-chain.md`](./07-scope-chain.md) — How scope works inside functions
- [`10-async-patterns.md`](./10-async-patterns.md) — Callbacks, promises, async/await
