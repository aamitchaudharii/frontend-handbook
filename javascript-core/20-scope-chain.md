# 07 — Scope Chain

> **"The scope chain is the path JavaScript walks to find a variable. Understanding it is understanding why code can see some things and not others — and why that behavior is determined by where you write code, not where you run it."**

Scope is one of the most fundamental concepts in JavaScript. Every bug involving "variable is not defined," every confusion about `this`, every question about why a value is unexpected inside a callback — traces back to scope. This document covers every type of scope, how the chain is formed, how lookup works, and the real-world implications for writing clean, predictable code.

---

## 📚 Table of Contents

1. [What Is Scope?](#1-what-is-scope)
2. [Lexical Scoping — The Core Rule](#2-lexical-scoping--the-core-rule)
3. [Types of Scope](#3-types-of-scope)
4. [The Scope Chain — How Lookup Works](#4-the-scope-chain--how-lookup-works)
5. [Global Scope](#5-global-scope)
6. [Function Scope](#6-function-scope)
7. [Block Scope (`let` and `const`)](#7-block-scope-let-and-const)
8. [Module Scope](#8-module-scope)
9. [The Temporal Dead Zone (TDZ)](#9-the-temporal-dead-zone-tdz)
10. [Variable Shadowing](#10-variable-shadowing)
11. [Scope Chain vs Prototype Chain](#11-scope-chain-vs-prototype-chain)
12. [Hoisting in the Context of Scope](#12-hoisting-in-the-context-of-scope)
13. [Scope in Loops](#13-scope-in-loops)
14. [IIFE and Scope Isolation](#14-iife-and-scope-isolation)
15. [Dynamic Scope — `eval` and `with`](#15-dynamic-scope--eval-and-with)
16. [Good Practices](#16-good-practices)
17. [Bad Practices](#17-bad-practices)
18. [Common Mistakes](#18-common-mistakes)
19. [Interview-Level Explanation](#19-interview-level-explanation)
20. [Exercises](#20-exercises)

---

## 1. What Is Scope?

Scope is the set of variables, functions, and objects accessible in a given part of your code. It defines:

- **Where** a variable is visible
- **How long** a variable lives
- **What** can access it

```javascript
const x = "outer";

function fn() {
  const y = "inner";
  console.log(x); // ✓ — x is in outer scope, accessible here
  console.log(y); // ✓ — y is in this scope
}

fn();
console.log(y); // ✗ — ReferenceError: y is not defined
//     y only exists inside fn's scope
```

Think of scope as **visibility rules** baked into the code's structure at write time.

---

## 2. Lexical Scoping — The Core Rule

JavaScript uses **lexical scoping** (also called static scoping). This means the scope of a variable is determined by **where it is written in the source code** — not where it is called or invoked at runtime.

```javascript
const name = "Global";

function outer() {
  const name = "Outer";

  function inner() {
    console.log(name); // Which `name`?
  }

  return inner;
}

const fn = outer();
fn(); // 'Outer' — NOT 'Global'
```

Why `'Outer'`? Because `inner` is **defined** inside `outer`. Its scope chain is established at write time — at the point where it appears in the source. When `inner` looks up `name`, it first checks its own scope (not found), then walks to its **lexically enclosing** scope — `outer` — where it finds `name = 'Outer'`.

Even though `fn()` is **called** from the global scope, it uses the scope chain from where it was **written**, not where it's called.

```
Lexical scope chain for inner:

  inner (defined inside outer)
    └── outer (enclosing scope)
          └── global (outer's enclosing scope)
```

This is the opposite of dynamic scoping (used in some older languages like Bash), where the scope at call time would determine lookup. JavaScript is always lexical.

---

## 3. Types of Scope

JavaScript has four scope types:

```
┌──────────────────────────────────────────────────────────┐
│                     SCOPE TYPES                           │
│                                                            │
│  1. Global Scope    — top-level, accessible everywhere    │
│  2. Function Scope  — inside a function, var-based        │
│  3. Block Scope     — inside {}, let/const-based          │
│  4. Module Scope    — per ES module file                  │
└──────────────────────────────────────────────────────────┘
```

Each creates a new **lexical environment** with its own **EnvironmentRecord** and a reference to its outer scope.

---

## 4. The Scope Chain — How Lookup Works

The scope chain is the linked list of lexical environments from the current scope outward to the global scope. Variable lookup walks this chain.

### Lookup Algorithm

```
To resolve identifier `x` in the current scope:

1. Check current scope's EnvironmentRecord
   → Found? Return value. Done.
   → Not found? Follow outer reference.

2. Check outer scope's EnvironmentRecord
   → Found? Return value. Done.
   → Not found? Follow its outer reference.

3. Continue until reaching global scope.
   → Found in global? Return value.
   → Not found in global? ReferenceError (or undefined for typeof)
```

### Visual Scope Chain

```javascript
const a = 1;

function outer() {
  const b = 2;

  function middle() {
    const c = 3;

    function inner() {
      const d = 4;
      // Can access: d (own), c (middle), b (outer), a (global)
      console.log(a + b + c + d); // 10
    }

    inner();
  }

  middle();
}

outer();
```

```
Scope chain when inner() executes:

inner's env:   { d: 4 }       outer ref ──►
middle's env:  { c: 3 }       outer ref ──►
outer's env:   { b: 2 }       outer ref ──►
global env:    { a: 1, outer: fn }   outer ref ──► null
```

Variable `a` requires walking 3 scopes outward. Variable `d` is found immediately in the first scope.

### The Outer Reference Is Set at Definition Time

This is the essence of lexical scoping:

```javascript
function makeEnv(val) {
  const env = val;
  return function read() {
    return env; // outer ref points to makeEnv's scope
  };
}

const readA = makeEnv("A");
const readB = makeEnv("B");

readA(); // 'A' — chain leads to makeEnv call 1's env
readB(); // 'B' — chain leads to makeEnv call 2's env

// readA and readB have DIFFERENT outer references
// even though they're the same function template
```

---

## 5. Global Scope

The global scope is the outermost scope. Variables defined here are accessible everywhere in the program.

### Browser Global Scope

```javascript
// Variables at the top level of a script (not module) become global
var globalVar = "I am global";
let blockGlobal = "also accessible globally, but not on window";
const CONST_GLOBAL = "same as let";

// var at global level attaches to window object
console.log(window.globalVar); // 'I am global'
console.log(window.blockGlobal); // undefined — let doesn't attach to window
```

### Node.js Global Scope

```javascript
// In Node.js, each file is a module — top-level is MODULE scope, not global
// To truly add to global scope in Node.js:
global.myGlobal = "everywhere"; // explicit global assignment

// Top-level vars are module-scoped (not global):
var localToModule = "only in this file"; // NOT on global object
```

### Global Scope Pollution — The Problem

```javascript
// ❌ Accidental global creation (no declaration keyword)
function calculate() {
  result = 42; // no var/let/const → creates global variable!
}

calculate();
console.log(result); // 42 — now a global
console.log(window.result); // 42 — pollutes window object

// In strict mode, this throws:
("use strict");
function calculate() {
  result = 42; // ReferenceError: result is not defined
}
```

### Avoiding Global Scope Pollution

```javascript
// ✅ Always declare variables
function calculate() {
  const result = 42; // local, not global
  return result;
}

// ✅ Use modules (automatic module scope)
// file: utils.js (ES module)
export function calculate() {
  const result = 42;
  return result;
}
// result is not global — it's scoped to the function
```

---

## 6. Function Scope

Variables declared with `var` inside a function are **function-scoped** — they're accessible anywhere within that function, but not outside.

```javascript
function example() {
  var x = 1;

  if (true) {
    var x = 2; // same variable! var is function-scoped, not block-scoped
    console.log(x); // 2
  }

  console.log(x); // 2 — the if block didn't create a new scope for var
}

console.log(x); // ReferenceError — x is function-scoped to example
```

### `var` Hoisting within Function Scope

`var` declarations are hoisted to the **top of their function scope** (or global scope), initialized to `undefined`:

```javascript
function hoistExample() {
  console.log(x); // undefined — hoisted to top of function
  console.log(y); // ReferenceError — let is in TDZ

  var x = 5; // declaration hoisted, assignment stays here
  let y = 10;
}
```

The engine treats this as:

```javascript
function hoistExample() {
  var x; // hoisted to top, initialized undefined
  // (y is in TDZ until its declaration line)

  console.log(x); // undefined
  console.log(y); // ReferenceError: Cannot access 'y' before initialization

  x = 5; // assignment stays in place
  let y = 10;
}
```

### Function Scope Creates Isolation

```javascript
// Each function call creates a new scope — isolated from other calls
function createId() {
  let id = Math.random().toString(36).slice(2);
  return id;
}

const id1 = createId(); // new scope, own `id`
const id2 = createId(); // new scope, own `id` — completely separate
```

---

## 7. Block Scope (`let` and `const`)

`let` and `const` are **block-scoped** — they're limited to the nearest enclosing `{}` block: `if`, `for`, `while`, `try/catch`, or any standalone `{}`.

```javascript
{
  let blockVar = "inside block";
  const BLOCK_CONST = "also inside";
  console.log(blockVar); // ✓
}
console.log(blockVar); // ReferenceError — out of block scope
```

### `let` vs `var` in Blocks

```javascript
// var — function scoped, bleeds out of blocks
for (var i = 0; i < 3; i++) {
  // i is in function scope (or global if no enclosing function)
}
console.log(i); // 3 — i leaked out of the for loop

// let — block scoped, contained in the for block
for (let j = 0; j < 3; j++) {
  // j exists only inside the for loop body
}
console.log(j); // ReferenceError — j doesn't exist here
```

### Block Scope in `if` Statements

```javascript
let status = "pending";

if (status === "pending") {
  let message = "Processing..."; // block-scoped to the if
  const urgency = "high"; // block-scoped to the if
  console.log(message); // ✓
}

console.log(message); // ReferenceError — out of scope
console.log(urgency); // ReferenceError — out of scope
```

### Standalone Blocks for Scope Isolation

```javascript
// You can use standalone blocks to isolate variables
{
  const temp = heavyComputation();
  processWith(temp);
  // temp is available only in this block
  // After this block, temp is eligible for GC
}
// temp is out of scope here — no reference held
```

---

## 8. Module Scope

ES Modules (files with `import`/`export`) create their own scope. Top-level declarations in a module are **module-scoped** — not global.

```javascript
// math.js (ES module)
const PI = 3.14159; // module-scoped — NOT global
let computeCount = 0; // module-scoped — private to this module

export function circleArea(r) {
  computeCount++; // can access module-scoped variable
  return PI * r * r;
}

export function getComputeCount() {
  return computeCount;
}
```

```javascript
// app.js
import { circleArea, getComputeCount } from "./math.js";

console.log(PI); // ReferenceError — PI is not exported, not global
circleArea(5);
circleArea(3);
getComputeCount(); // 2 — module-scoped state preserved between calls
```

### Module Scope Benefits

- No global pollution — top-level vars stay in the module
- Explicit API surface — only exported items are accessible
- Module-level state (like `computeCount`) is private by default
- Enables tree-shaking — bundlers can eliminate unused exports

### Module Scope vs Script Scope

```html
<!-- Script (no type="module") — top-level vars are global -->
<script>
  var message = "global"; // window.message = 'global'
</script>

<!-- Module — top-level vars are module-scoped -->
<script type="module">
  var message = "module"; // NOT on window — module scope
  console.log(window.message); // undefined
</script>
```

---

## 9. The Temporal Dead Zone (TDZ)

`let` and `const` are hoisted like `var`, but they are **not initialized** until their declaration line is reached. The period between the start of the block and the declaration is called the **Temporal Dead Zone (TDZ)**.

```javascript
{
  // TDZ for `x` starts here
  console.log(x); // ReferenceError: Cannot access 'x' before initialization
  // NOT "undefined" like var would be
  let x = 5; // TDZ ends here — x is now initialized
  console.log(x); // 5
}
```

### Why TDZ Exists

TDZ was introduced to catch programming errors. With `var`, accessing before declaration silently gives `undefined` — which is a common source of bugs. With `let`/`const`, you get an immediate error, making the bug obvious.

```javascript
// var: silent bug
function bugWithVar() {
  console.log(config); // undefined — no error, confusing behavior
  // ... many lines later ...
  var config = { debug: true };
}

// let: immediate, obvious error
function clearWithLet() {
  console.log(config); // ReferenceError — clear and immediate
  // ... many lines later ...
  let config = { debug: true };
}
```

### TDZ with Function Parameters

```javascript
// Default parameters have their own scope and can reference each other
function example(a = 1, b = a + 1) {
  // b defaults to a + 1 — works because a is initialized before b
  return a + b;
}

// But you can't reference a later parameter from an earlier one
function broken(a = b, b = 2) {
  // ReferenceError: b is in TDZ when a is evaluated
  return a + b;
}
```

---

## 10. Variable Shadowing

A variable in an inner scope with the same name as a variable in an outer scope **shadows** the outer variable — the inner scope sees only its own binding.

```javascript
const color = "blue"; // outer

function paintRoom() {
  const color = "red"; // inner — shadows outer `color`
  console.log(color); // 'red' — inner takes precedence
}

paintRoom();
console.log(color); // 'blue' — outer unchanged
```

### Intentional vs Accidental Shadowing

```javascript
// ✅ Intentional shadowing — often useful in callbacks
const items = [1, 2, 3];
items.forEach((item) => {
  // `item` shadows nothing — fresh name
  console.log(item);
});

// ✅ Intentional — narrow scope for temp variable
function processData(data) {
  const result = transform(data);
  {
    const result = validate(result); // different result in inner block
    if (!result.valid) throw new Error(result.error);
  }
  return result; // uses the outer result
}

// ❌ Accidental shadowing — confusing, error-prone
function confusing() {
  let count = 0;
  function increment() {
    let count = 1; // accidentally shadows outer count
    return count; // always 1, never modifies outer count
  }
  increment();
  console.log(count); // 0 — increment did nothing to outer count
}
```

### Shadowing Parameters

```javascript
function tricky(x) {
  // x is a parameter
  if (x > 10) {
    let x = x * 2; // ❌ TDZ! — `let x` shadows the parameter
    // but is in TDZ at the point of `x * 2`
    // ReferenceError: Cannot access 'x' before initialization
    return x;
  }
  return x;
}
```

---

## 11. Scope Chain vs Prototype Chain

These two chains are often confused because both are lookup mechanisms. They are completely separate:

```
SCOPE CHAIN:                      PROTOTYPE CHAIN:
Used for: variable lookup         Used for: property lookup on objects
Traversed: at variable access     Traversed: at property access
Links: lexical environments       Links: [[Prototype]] references
Direction: outward from current   Direction: up from instance to Object.prototype
Determined: at write time         Determined: at object creation
```

```javascript
const x = 10; // scope chain lookup

const obj = { x: 10 }; // prototype chain lookup when accessing obj.x

function fn() {
  console.log(x); // ← scope chain lookup
  console.log(obj.x); // ← property lookup on obj (prototype chain if not own)
}
```

### They Don't Mix

```javascript
// Scope chain does NOT traverse into objects
const config = { debug: true };

function check() {
  console.log(debug); // ❌ ReferenceError — scope chain looks for `debug`
  // It does NOT look inside `config` object
}

// You must explicitly access the property:
function check() {
  console.log(config.debug); // ✓ property access on config
}
```

---

## 12. Hoisting in the Context of Scope

Hoisting is the result of the **creation phase** running before execution within each scope. It has different behavior based on declaration type:

```
Variable/Function    Hoisted?    Initialized?    Scope
─────────────────────────────────────────────────────────
var                  ✅ Yes       undefined       Function/Global
let                  ✅ Yes       ❌ TDZ          Block
const                ✅ Yes       ❌ TDZ          Block
function declaration ✅ Yes       Full function   Function/Global
function expression  ✅ (var)     undefined       Function/Global
arrow function       ✅ (if var)  undefined       Block (if let/const)
class                ✅ Yes       ❌ TDZ          Block
```

### Function Declarations Are Fully Hoisted

```javascript
// ✅ Can call before declaration — fully hoisted
console.log(greet("Alice")); // 'Hello Alice'

function greet(name) {
  return `Hello ${name}`;
}
```

### Function Expressions Are Not Fully Hoisted

```javascript
// ❌ Cannot call before declaration
console.log(greet("Alice")); // TypeError: greet is not a function
// greet is hoisted as undefined (var)

var greet = function (name) {
  return `Hello ${name}`;
};
```

### Class Declarations Are in TDZ

```javascript
const instance = new MyClass(); // ReferenceError: Cannot access 'MyClass' before initialization

class MyClass {
  constructor() {
    this.x = 1;
  }
}
```

---

## 13. Scope in Loops

Loops are a source of classic scoping bugs. The interaction between `var`, `let`, and closures in loops is critical to understand.

### `var` in a Loop — One Binding

```javascript
// ONE `i` variable — shared across all iterations and all closures
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Output: 3, 3, 3 (after loop finishes, i = 3)
```

### `let` in a Loop — New Binding Per Iteration

```javascript
// NEW `i` for each iteration — each closure gets its own
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Output: 0, 1, 2

// Equivalent to what the engine creates:
{
  let i = 0;
  setTimeout(() => console.log(i), 0); // closes over i = 0
}
{
  let i = 1;
  setTimeout(() => console.log(i), 0); // closes over i = 1
}
{
  let i = 2;
  setTimeout(() => console.log(i), 0); // closes over i = 2
}
```

### `const` in `for...of` Loops

```javascript
// ✅ const works in for...of — new binding per iteration (not incremented)
const items = ["a", "b", "c"];
for (const item of items) {
  setTimeout(() => console.log(item), 0);
}
// Output: a, b, c — each iteration has its own const binding

// ❌ const doesn't work in regular for loop — reassignment attempted
for (const i = 0; i < 3; i++) {
  // TypeError: Assignment to constant variable
  // i++ tries to reassign `i`
}
```

### Scope Isolation for Async Loop Work

```javascript
// ❌ All fetches share same `i` (var)
for (var i = 0; i < users.length; i++) {
  fetch(`/api/users/${users[i].id}`)
    .then((r) => r.json())
    .then((data) => {
      console.log(`User ${i}:`, data); // i is always users.length here
    });
}

// ✅ let gives each iteration its own i
for (let i = 0; i < users.length; i++) {
  fetch(`/api/users/${users[i].id}`)
    .then((r) => r.json())
    .then((data) => {
      console.log(`User ${i}:`, data); // correct i captured
    });
}

// ✅✅ Even better: for...of with destructuring
for (const user of users) {
  fetch(`/api/users/${user.id}`)
    .then((r) => r.json())
    .then((data) => {
      console.log(`User ${user.id}:`, data); // user captured per iteration
    });
}
```

---

## 14. IIFE and Scope Isolation

The **Immediately Invoked Function Expression (IIFE)** was the pre-`let`/`const` way to create isolated scopes. It's still useful today in specific contexts.

### Classic IIFE Pattern

```javascript
// Create an isolated scope — nothing leaks to outer scope
(function () {
  var privateVar = "only here";
  function privateHelper() {
    /* ... */
  }

  // Expose only what's needed
  window.myLib = {
    doThing: privateHelper,
  };
})();

console.log(privateVar); // ReferenceError — not accessible outside
```

### IIFE for Module Pattern (Pre-ES Modules)

```javascript
const Calculator = (function () {
  let history = []; // private

  function record(operation, result) {
    history.push({ operation, result, time: Date.now() });
  }

  return {
    add(a, b) {
      const result = a + b;
      record(`${a} + ${b}`, result);
      return result;
    },
    subtract(a, b) {
      const result = a - b;
      record(`${a} - ${b}`, result);
      return result;
    },
    getHistory() {
      return [...history]; // return copy
    },
  };
})();

Calculator.add(1, 2); // 3
Calculator.getHistory(); // [{ operation: '1 + 2', result: 3, ... }]
Calculator.history; // undefined — private
```

### When to Use IIFE Today

- Wrapping legacy code that uses `var` to prevent global leaks
- Creating a new scope around `await` at the top level (before top-level await)
- Immediately running initialization logic with a clear isolated scope

```javascript
// Top-level await alternative (where not supported)
(async () => {
  const config = await fetchConfig();
  initApp(config);
})();
```

---

## 15. Dynamic Scope — `eval` and `with`

JavaScript is lexically scoped, but two mechanisms introduce dynamic-like scope behavior. Both are strongly discouraged.

### `eval` — Runtime Scope Injection

```javascript
function dangerous(code) {
  const localVar = 42;
  eval(code); // code can see and modify localVar!
}

dangerous("console.log(localVar)"); // 42 — eval injects into current scope
dangerous("localVar = 99"); // modifies localVar — security nightmare
```

**Why `eval` is bad for scope:**

- Prevents V8 from static analysis of the scope — disables all scope optimizations
- Any function containing `eval` is treated as "may have dynamic scope" → V8 can't use hidden classes effectively
- Security: any string becomes executable code

### `with` — Object Properties as Scope

```javascript
// ❌ Banned in strict mode
const obj = { x: 1, y: 2 };

with (obj) {
  console.log(x + y); // x and y resolved as obj.x and obj.y
  // But x could also be a variable in outer scope — ambiguous!
}
```

`with` makes scope unpredictable. The engine can't know at compile time whether `x` refers to `obj.x` or an outer `x`. Completely banned in strict mode (which is always enabled in classes and modules).

---

## 16. Good Practices

### ✅ Always use `let` and `const` — never `var`

```javascript
// ✅ Block-scoped, predictable, TDZ protects against use-before-declare
const MAX_SIZE = 100; // immutable binding
let currentSize = 0; // mutable binding

// ❌ Function-scoped, hoisted to undefined, re-declarable
var count = 0;
var count = 5; // silently re-declares — no error
```

### ✅ Declare variables as close to use as possible

```javascript
// ❌ Declaration at top, use far below — wide scope
function process(items) {
  let filtered; // declared here
  let transformed; // declared here
  let result; // declared here

  // 30 lines of code...

  filtered = items.filter(isValid);
  transformed = filtered.map(transform);
  result = transformed.reduce(accumulate, []);
  return result;
}

// ✅ Declare where first used — narrow scope, easier to read
function process(items) {
  // 30 lines of code (that don't need filtered/transformed/result)...

  const filtered = items.filter(isValid);
  const transformed = filtered.map(transform);
  const result = transformed.reduce(accumulate, []);
  return result;
}
```

### ✅ Use modules for scope isolation

```javascript
// ✅ Module scope is naturally isolated — no IIFE needed
// utils.js
const cache = new Map(); // module-private, not global

export function getCached(key) {
  return cache.get(key);
}

export function setCached(key, value) {
  cache.set(key, value);
}
```

### ✅ Use `const` by default, `let` when you need to reassign

```javascript
// ✅ const signals intent: this binding won't change
const user = { name: "Alice" }; // const for the binding
user.name = "Bob"; // object itself can still be mutated

// ✅ let for values you'll reassign
let retries = 0;
while (retries < 3) {
  retries++;
}
```

### ✅ Avoid variable shadowing — use distinct names

```javascript
// ❌ Confusing — inner `data` shadows outer `data`
async function load(url) {
  const data = await fetch(url);
  return data.json().then((data) => {
    // inner `data` shadows outer
    return process(data);
  });
}

// ✅ Use distinct names
async function load(url) {
  const response = await fetch(url);
  return response.json().then((parsed) => {
    return process(parsed);
  });
}
```

---

## 17. Bad Practices

### ❌ Relying on `var` hoisting

```javascript
// ❌ Relying on var being hoisted to undefined
function example() {
  if (shouldInitialize) {
    var config = loadConfig(); // relying on hoisting behavior
  }
  return config; // could be undefined — confusing
}
```

### ❌ Global scope pollution

```javascript
// ❌ Implicit global creation
function init() {
  apiKey = process.env.API_KEY; // no const/let/var → global pollution
}
```

### ❌ Excessively large scope

```javascript
// ❌ `result` accessible in entire function even after its job is done
function process(data) {
  const result = computeHeavyResult(data); // large object
  renderResult(result);
  // result stays in scope for the rest of the function
  // 100 more lines of code...
  sendAnalytics(data); // result still alive in scope
}

// ✅ Isolate result to where it's needed
function process(data) {
  {
    const result = computeHeavyResult(data);
    renderResult(result);
    // result goes out of scope here, eligible for GC
  }
  // 100 more lines of code... result is gone
  sendAnalytics(data);
}
```

### ❌ Using `eval` or `with`

```javascript
// ❌ Never
eval(userInput); // security hole + scope destroyer
with (obj) {
  /* ambiguous */
} // banned in strict mode
```

---

## 18. Common Mistakes

### Mistake 1 — Thinking block scope applies to `var`

```javascript
if (true) {
  var x = 10; // var is NOT block-scoped
}
console.log(x); // 10 — var leaked out of the if block
```

### Mistake 2 — Accessing `let`/`const` before declaration

```javascript
function example() {
  console.log(name); // ReferenceError (TDZ), NOT undefined
  let name = "Alice";
}
// Many developers expect `undefined` like var — that's wrong for let/const
```

### Mistake 3 — Shadowing function parameters with block variables

```javascript
function tricky(value) {
  if (value > 0) {
    let value = value * 2; // ❌ ReferenceError — TDZ!
    // `let value` is hoisted to TDZ, so `value * 2` tries to read
    // the TDZ `value`, not the parameter
    return value;
  }
  return value;
}

// Fix: use a different name
function tricky(value) {
  if (value > 0) {
    const doubled = value * 2; // different name — no shadowing
    return doubled;
  }
  return value;
}
```

### Mistake 4 — Scope and `this` are different things

```javascript
const obj = {
  name: "Alice",
  getName: function () {
    // `this` is determined by call site, not scope chain
    return this.name; // works when called as obj.getName()
  },
  getNameArrow: () => {
    // Arrow function: `this` is from LEXICAL scope
    // But scope chain doesn't contain `name` variable — only `this` does
    return this.name; // undefined — `this` = window/global in arrow fn
  },
};
```

---

## 19. Interview-Level Explanation

> **"What is the scope chain in JavaScript? How does variable lookup work?"**

**Strong answer:**

> "The scope chain is the linked structure of lexical environments that JavaScript walks when it needs to resolve a variable name. Every function creates a new lexical environment with its own variable bindings, and each environment holds a reference to its outer environment — the scope in which that function was _defined_. This forms a chain from the current scope outward to the global scope.
>
> When JavaScript encounters an identifier, it first looks in the current scope's EnvironmentRecord. If not found, it follows the outer reference to the next scope and checks there. This continues until either the variable is found or the chain ends at null (after the global scope), at which point a ReferenceError is thrown.
>
> The critical word is _lexical_ — the outer reference is established at write time, not call time. A function's scope chain is determined by where it appears in the source code, not where it's invoked. This is why a closure can access variables from its enclosing function even after that function has returned — the outer reference in the chain still points to the environment record in the heap.
>
> JavaScript has four scope types: global scope (accessible everywhere), function scope (var declarations are scoped to the nearest function), block scope (let and const are scoped to the nearest {} block), and module scope (top-level declarations in ES modules are private to the module). The shift from var to let/const moved JavaScript from primarily function scope to primarily block scope — more predictable, less bug-prone, and closer to how most other languages work."

---

## 20. Exercises

### Exercise 1 — Predict scope behavior

```javascript
let x = "global";

function outer() {
  let x = "outer";

  function inner() {
    console.log(x); // ?
  }

  let x = "outer-redeclared"; // ← what happens here?

  inner();
}

outer();
```

<details>
<summary>Answer</summary>

```
This code throws: SyntaxError: Identifier 'x' has already been declared

`let x = 'outer'` and `let x = 'outer-redeclared'` are both in the
same function scope. `let` cannot be re-declared in the same scope.

If the second declaration were removed:
  inner() would log 'outer' — found in outer's scope via chain
```

</details>

---

### Exercise 2 — Block scope isolation

```javascript
let result;

{
  const a = 10;
  const b = 20;
  result = a + b;
}

console.log(result); // ?
console.log(a); // ?
console.log(b); // ?
```

<details>
<summary>Answer</summary>

```
result → 30  ✓ — result is in outer scope, assigned inside block
a      → ReferenceError — a is block-scoped to the {} block
b      → ReferenceError — b is block-scoped to the {} block
```

</details>

---

### Exercise 3 — TDZ detection

Which of these throw errors, and what kind?

```javascript
// a)
console.log(typeof undeclaredVar);

// b)
console.log(typeof myLet);
let myLet = 5;

// c)
console.log(myConst);
const myConst = 5;

// d)
{
  console.log(innerLet);
  let innerLet = 5;
}
```

<details>
<summary>Answer</summary>

```
a) typeof undeclaredVar → "undefined"
   (typeof is special-cased: does NOT throw for undeclared vars)

b) typeof myLet → ReferenceError: Cannot access 'myLet' before initialization
   (typeof DOES throw for TDZ variables — a common misconception that typeof
    is always safe; it's only safe for completely undeclared variables)

c) console.log(myConst) → ReferenceError: Cannot access 'myConst' before initialization
   (TDZ for const)

d) console.log(innerLet) → ReferenceError: Cannot access 'innerLet' before initialization
   (TDZ for let — block scope TDZ starts at beginning of the block)
```

</details>

---

### Exercise 4 — Build a scope-isolated feature manager

Implement a feature flag system where:

- Flags are module-private (not accessible from outside)
- `enable(flag)` / `disable(flag)` modify flags
- `isEnabled(flag)` checks a flag
- `withFlag(flag, fn)` executes `fn` only if flag is enabled

```javascript
// Expected usage:
featureFlags.enable("darkMode");
featureFlags.isEnabled("darkMode"); // true
featureFlags.withFlag("darkMode", () => applyDarkTheme());
featureFlags.isEnabled("undeclaredFlag"); // false (not throws)
```

<details>
<summary>Solution</summary>

```javascript
const featureFlags = (function () {
  // Private scope — not accessible from outside
  const flags = new Set();
  const listeners = new Map();

  return {
    enable(flag) {
      flags.add(flag);
      listeners.get(flag)?.forEach((fn) => fn(true));
      return this;
    },

    disable(flag) {
      flags.delete(flag);
      listeners.get(flag)?.forEach((fn) => fn(false));
      return this;
    },

    isEnabled(flag) {
      return flags.has(flag);
    },

    withFlag(flag, fn) {
      if (flags.has(flag)) fn();
      return this;
    },

    onChange(flag, fn) {
      if (!listeners.has(flag)) listeners.set(flag, new Set());
      listeners.get(flag).add(fn);
      return () => listeners.get(flag).delete(fn); // unsubscribe
    },
  };
})();

// Usage
featureFlags.enable("darkMode");
featureFlags.withFlag("darkMode", () => console.log("Dark mode active!"));
featureFlags.isEnabled("darkMode"); // true
featureFlags.isEnabled("betaFeature"); // false

// flags and listeners are private — inaccessible
featureFlags.flags; // undefined
featureFlags.listeners; // undefined
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/01-execution-context.md`](./01-execution-context.md) — Lexical environments in detail
- [`javascript-core/05-closures.md`](./05-closures.md) — Closures and the scope chain in action
- [`javascript-core/06-prototypes.md`](./06-prototypes.md) — Prototype chain vs scope chain
- [`javascript-core/08-memory-management.md`](./08-memory-management.md) — How scope affects memory retention

---

<div align="center">

**Next:** [`javascript-core/08-memory-management.md`](./08-memory-management.md) →

</div>
