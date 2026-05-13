# 01 — Execution Context

> **"Every time JavaScript runs code, it builds an invisible structure around it — the execution context. Understanding this structure explains variable hoisting, `this` binding, closures, and scope — all at once."**

The execution context is the foundational concept of JavaScript. Everything else — closures, the call stack, `this`, hoisting, the event loop — is built on top of it. If you understand execution contexts deeply, dozens of confusing JavaScript behaviors become obvious.

---

## 📚 Table of Contents

1. [What Is an Execution Context?](#1-what-is-an-execution-context)
2. [Types of Execution Contexts](#2-types-of-execution-contexts)
3. [Phases of Execution Context Creation](#3-phases-of-execution-context-creation)
4. [The Variable Environment](#4-the-variable-environment)
5. [The Lexical Environment](#5-the-lexical-environment)
6. [The `this` Binding](#6-the-this-binding)
7. [Hoisting — Explained by Execution Context](#7-hoisting--explained-by-execution-context)
8. [Execution Context & the Call Stack](#8-execution-context--the-call-stack)
9. [Block Scope and `let`/`const`](#9-block-scope-and-letconst)
10. [Good Practices](#10-good-practices)
11. [Bad Practices](#11-bad-practices)
12. [Common Mistakes](#12-common-mistakes)
13. [Interview-Level Explanation](#13-interview-level-explanation)
14. [Exercises](#14-exercises)

---

## 1. What Is an Execution Context?

When JavaScript runs any piece of code — a script, a function call, an `eval` — the engine creates an **execution context**: a wrapper that contains everything needed to execute that code.

Think of it as the "environment" for a block of code. It defines:

- Which variables are accessible
- What `this` refers to
- Where to look for outer variables (the scope chain)

```
┌─────────────────────────────────────────────────────────────┐
│                    EXECUTION CONTEXT                         │
│                                                              │
│  ┌──────────────────────┐   ┌──────────────────────────┐   │
│  │  Variable Environment│   │   Lexical Environment     │   │
│  │  (var declarations,  │   │   (let/const, functions,  │   │
│  │   function decls)    │   │    outer scope ref)       │   │
│  └──────────────────────┘   └──────────────────────────┘   │
│                                                              │
│  ┌──────────────────────┐                                   │
│  │   `this` Binding     │                                   │
│  │   (depends on how    │                                   │
│  │    code was called)  │                                   │
│  └──────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Types of Execution Contexts

There are exactly three types:

### 2.1 Global Execution Context (GEC)

- Created **once** when the script first runs
- Represents the top-level code (not inside any function)
- Creates the global object (`window` in browsers, `global` in Node.js)
- Sets `this` to the global object
- Only **one** GEC exists per JS runtime

```javascript
// This code runs in the Global Execution Context
var name = 'Alice';       // added to global object: window.name
function greet() { ... }  // added to global object: window.greet
```

### 2.2 Function Execution Context (FEC)

- Created **every time** a function is **called** (not defined)
- Each call creates a **new, fresh** context — even recursive calls
- Destroyed when the function returns (popped from call stack)
- Can be thousands at once (nested calls)

```javascript
function add(a, b) {
  // New FEC created when add() is called
  // a, b are in this context's variable environment
  return a + b;
  // FEC destroyed here
}

add(1, 2); // creates FEC #1
add(3, 4); // creates FEC #2 — completely separate from #1
```

### 2.3 Eval Execution Context

- Created when code is run via `eval()`
- Rarely used in modern JavaScript
- Has its own variable environment
- Not covered further — `eval()` is considered harmful in production

---

## 3. Phases of Execution Context Creation

Every execution context is created in **two phases**. This two-phase process is what causes hoisting.

### Phase 1: Creation Phase (Memory Phase)

Before a single line of code runs, the engine scans through the code and:

1. Creates the **Variable Environment** and **Lexical Environment**
2. For `var` declarations: creates the variable and sets it to `undefined`
3. For `function` declarations: stores the **entire function definition** in memory
4. For `let`/`const` declarations: creates the variable but leaves it **uninitialized** (Temporal Dead Zone)
5. Determines the `this` binding

```javascript
// What you write:
console.log(x); // line 1
var x = 10; // line 2
console.log(greet); // line 3
function greet() {
  return "hello";
} // line 4

// What the engine sees BEFORE running line 1 (creation phase):
// Memory layout after creation phase:
// x     → undefined     (var: allocated, set to undefined)
// greet → function() {  (function decl: entire body stored)
//           return 'hello'
//         }
```

### Phase 2: Execution Phase

Now the engine runs the code top to bottom:

```javascript
console.log(x); // undefined (created in phase 1, not yet assigned)
var x = 10; // NOW x is assigned 10
console.log(greet); // ƒ greet() — full function was stored in phase 1
function greet() {
  return "hello";
} // already processed — no-op during execution
```

### Two-Phase Visualization

```
CREATION PHASE (scan):           EXECUTION PHASE (run):
─────────────────────            ─────────────────────
x     = undefined         →      console.log(x)  → undefined
greet = ƒ(){...}          →      x = 10
                           →      console.log(greet) → ƒ greet
                           →      (function declaration — skipped)
```

---

## 4. The Variable Environment

The Variable Environment (VE) is a record within the execution context that stores all variable bindings for that context.

For the **Global Execution Context**:

- All `var` declarations become properties of the global object
- All function declarations become properties of the global object

```javascript
var age = 25;
function sayHi() {
  return "hi";
}

// In browser:
console.log(window.age); // 25
console.log(window.sayHi); // ƒ sayHi

// let/const do NOT become global object properties
let count = 0;
console.log(window.count); // undefined — stored in lexical env, not window
```

For **Function Execution Contexts**:

- Parameters are stored here
- Local `var` declarations are stored here
- Local function declarations are stored here

```javascript
function process(data, multiplier) {
  // Variable Environment for this FEC:
  // data       → (value of argument)
  // multiplier → (value of argument)
  // result     → undefined (hoisted var)
  // helper     → ƒ helper(){...} (hoisted function)

  var result = data * multiplier;

  function helper() {
    return result;
  }

  return helper();
}
```

---

## 5. The Lexical Environment

The Lexical Environment (LE) is similar to the Variable Environment but also contains a **reference to the outer lexical environment** — the environment of the code that _contains_ this function.

This outer reference is what creates the **scope chain**.

```
SCOPE CHAIN via Lexical Environment:

function outer() {
  let x = 10;

  function inner() {      ← inner's Lexical Env
    let y = 20;           │  y = 20
    console.log(x + y);  │  outer ref ──→ outer's Lexical Env
  }                                          x = 10
                                             outer ref ──→ Global LE
  inner();                                                   (no more outer)
}
```

When `inner` tries to access `x`:

1. Look in inner's LE → not found
2. Follow outer reference → outer's LE → found `x = 10` ✅

This chain of outer references is the **scope chain**. It's fixed at the time the function is **defined** (not called) — this is what "lexical" means.

```javascript
const x = "global";

function outer() {
  const x = "outer";

  function inner() {
    // inner's outer ref points to outer's LE (where x = 'outer')
    console.log(x); // 'outer' — follows scope chain to outer, finds x there
  }

  return inner;
}

const fn = outer();
fn(); // 'outer' — even though called from global scope
// Lexical environment is fixed at definition time, not call time
```

---

## 6. The `this` Binding

`this` is determined in the creation phase of the execution context — but its value depends entirely on **how the function is called**, not where it's defined (with the exception of arrow functions).

### `this` in the Global Context

```javascript
// Browser:
console.log(this); // window

// Node.js (module scope):
console.log(this); // {} (module.exports object)
```

### `this` in Function Calls

```javascript
function showThis() {
  console.log(this);
}

showThis(); // window (sloppy mode) or undefined (strict mode)

const obj = { showThis };
obj.showThis(); // obj — method call, this = the calling object

const bound = showThis.bind({ custom: true });
bound(); // { custom: true } — explicit binding wins

new showThis(); // newly created object — constructor call
```

### `this` Rules — Priority Order

```
Priority (highest to lowest):

1. new binding          new Fn()         → newly created object
2. Explicit binding     fn.call(ctx)     → ctx
                        fn.apply(ctx)    → ctx
                        fn.bind(ctx)()   → ctx
3. Implicit binding     obj.fn()         → obj
4. Default binding      fn()             → window / undefined (strict)
```

### Arrow Functions and `this`

Arrow functions do **not** create their own execution context for `this`. They inherit `this` from the enclosing lexical context at the time they are defined.

```javascript
class Timer {
  constructor() {
    this.seconds = 0;
  }

  start() {
    // ❌ Regular function — 'this' is undefined in strict mode
    // setInterval(function() {
    //   this.seconds++; // TypeError: Cannot set properties of undefined
    // }, 1000);

    // ✅ Arrow function — 'this' inherited from start()'s context = Timer instance
    setInterval(() => {
      this.seconds++; // works correctly
    }, 1000);
  }
}
```

Arrow functions **capture `this` from the enclosing execution context** at definition time. They don't have their own `this` binding in their execution context.

---

## 7. Hoisting — Explained by Execution Context

Hoisting is not a special language feature — it's a **side effect of the two-phase execution context creation**.

During the creation phase, the engine allocates memory for all declarations before any code runs. This makes declarations appear to "move to the top."

### `var` Hoisting

```javascript
console.log(a); // undefined — NOT ReferenceError
var a = 5;
console.log(a); // 5

// Why: creation phase set a = undefined
// Execution phase ran console.log(a) → undefined, then a = 5
```

### Function Declaration Hoisting

```javascript
greet(); // 'Hello!' — works before the declaration

function greet() {
  return "Hello!";
}

// Why: creation phase stored entire greet function in memory
// So it's available on line 1 of the execution phase
```

### `let` and `const` — Temporal Dead Zone (TDZ)

```javascript
console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 5;

// Why: creation phase created the binding for b but left it UNINITIALIZED
// Accessing an uninitialized binding = TDZ error
// The TDZ exists from the start of the block until the let/const line is executed
```

### Function Expression Hoisting

```javascript
greet(); // TypeError: greet is not a function

var greet = function () {
  return "Hello!";
};

// Why: `greet` is a var — hoisted to undefined
// Calling undefined() = TypeError
// (Not a ReferenceError — the name exists, but its value is undefined)
```

### Hoisting Priority

```javascript
var foo = 1;

function foo() {
  // function declaration hoisted ABOVE var
  return "function";
}

console.log(foo); // 1 — var assignment wins over function decl in execution phase

// Creation phase order:
// 1. Function declarations hoisted first (foo = ƒ)
// 2. var declarations hoisted (foo already exists — var doesn't overwrite)
// Execution phase:
// foo = 1 (var assignment runs, overwriting the function)
```

---

## 8. Execution Context & the Call Stack

Each time a function is called, its execution context is pushed onto the **call stack**. When it returns, it's popped.

```javascript
function multiply(a, b) {
  return a * b; // EC3 created/destroyed
}

function square(n) {
  return multiply(n, n); // EC2 created, calls multiply
}

function printSquare(n) {
  const result = square(n); // EC1 created, calls square
  console.log(result);
}

printSquare(4);
```

```
Call Stack evolution:

Initial:          After printSquare:   After square:       After multiply:
┌──────────┐     ┌──────────────────┐ ┌────────────────┐  ┌─────────────┐
│          │     │ printSquare EC   │ │ square EC      │  │ multiply EC │
│  Global  │     │ printSquare EC   │ │ printSquare EC │  │ square EC   │
│   EC     │     │ Global EC        │ │ Global EC      │  │ printSquare │
└──────────┘     └──────────────────┘ └────────────────┘  │ Global EC   │
                                                            └─────────────┘

After multiply returns: multiply EC popped
After square returns:   square EC popped
After printSquare ret:  printSquare EC popped
Final: only Global EC remains
```

Each execution context on the stack is completely **isolated** — `multiply`'s local variables don't exist in `square`'s context and vice versa.

---

## 9. Block Scope and `let`/`const`

ES6 introduced **block-scoped** declarations with `let` and `const`. These create a new Lexical Environment for each block `{}` — not a new execution context.

```javascript
function example() {
  // Function-level Lexical Env:
  let a = 1;

  {
    // Block-level Lexical Env (new env, outer ref → function env):
    let b = 2;
    console.log(a); // 1 — found via outer reference
    console.log(b); // 2 — found in block env
  }

  // console.log(b); // ReferenceError — b's env no longer accessible
}
```

### `var` vs `let`/`const` — Scope Difference

```javascript
// var: function-scoped (stored in function's Variable Environment)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3 3 3
// Why: ONE var i shared across all iterations (stored in FEC)
// All closures reference the same i — by the time they run, i = 3

// let: block-scoped (new Lexical Env per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0 1 2
// Why: NEW let i per iteration (new block env per loop body)
// Each closure captures a different i
```

---

## 10. Good Practices

### ✅ Use `const` by default, `let` when reassignment needed, avoid `var`

```javascript
// ✅ Block-scoped, predictable, no hoisting surprises
const MAX_RETRIES = 3;
let currentRetry = 0;

// ❌ Function-scoped, hoisted to undefined, confusing in loops
var retryCount = 0;
```

### ✅ Declare variables at the top of their scope

```javascript
// ✅ Makes scope obvious, mirrors what the engine does anyway
function processOrder(order) {
  const tax = order.price * 0.1;
  const total = order.price + tax;
  const orderId = generateId();
  // ... rest of function uses these
}
```

### ✅ Understand `this` before using it — prefer explicit binding

```javascript
// ✅ Arrow function captures this from class context
class EventHandler {
  constructor(element) {
    this.element = element;
    // Arrow: this = EventHandler instance
    this.element.addEventListener("click", () => this.handleClick());
  }
  handleClick() {
    console.log("clicked", this.element);
  }
}
```

### ✅ Use IIFE for isolated execution contexts

```javascript
// ✅ IIFE creates its own EC — variables don't pollute outer scope
(function () {
  const privateVar = "not accessible outside";
  // ... setup code ...
})();

// Modern equivalent: block with let/const
{
  const privateVar = "not accessible outside";
}
```

---

## 11. Bad Practices

### ❌ Relying on `var` hoisting

```javascript
// ❌ Confusing — using a variable before it's declared
function process() {
  if (condition) {
    result = compute(); // var hoisted — doesn't throw, but misleading
  }
  var result;
  return result;
}
```

### ❌ Implicit global variables

```javascript
// ❌ Assigning without declaration → creates global variable (sloppy mode)
function calculate() {
  total = 100; // no var/let/const → goes on window.total
}
// Use 'use strict' or always declare variables
```

### ❌ `this` inside nested regular functions

```javascript
// ❌ `this` in nested function is not the outer `this`
class Component {
  fetchData() {
    fetch("/api").then(function (response) {
      this.data = response; // TypeError: 'this' is undefined (strict mode)
    });
  }
}

// ✅ Fix: use arrow function or bind
class Component {
  fetchData() {
    fetch("/api").then((response) => {
      this.data = response; // arrow: this = Component instance
    });
  }
}
```

---

## 12. Common Mistakes

### Mistake 1 — Expecting `let` inside blocks to be accessible outside

```javascript
if (true) {
  let blockVar = "inside";
  var funcVar = "also inside";
}
console.log(blockVar); // ReferenceError — block scope
console.log(funcVar); // 'also inside' — function scope
```

### Mistake 2 — Thinking function declarations are re-evaluated each call

```javascript
function outer() {
  function inner() {
    // declared fresh each call? NO — hoisted once
    return "inner";
  }
  return inner;
}
// inner is created once in outer's creation phase,
// not re-created on every call to outer
```

### Mistake 3 — Losing `this` when destructuring methods

```javascript
const obj = {
  name: "Alice",
  greet() {
    return `Hello, ${this.name}`;
  },
};

const { greet } = obj;
greet(); // TypeError or 'Hello, undefined'
// Destructuring extracts the function — this binding is lost
// this is now global/undefined in strict mode

// Fix: bind it
const greet = obj.greet.bind(obj);
```

---

## 13. Interview-Level Explanation

> **"What is an execution context in JavaScript?"**

**Strong answer:**

> "An execution context is the environment the JavaScript engine creates to run a piece of code. Every function call and the global script create their own execution context.
>
> Each context has three main components: a Variable Environment that stores `var` declarations and function declarations for that scope, a Lexical Environment that stores `let`/`const` declarations and crucially holds a reference to the outer lexical environment — creating the scope chain — and a `this` binding determined by how the code was invoked.
>
> Execution contexts are created in two phases. In the creation phase, the engine scans the code and allocates memory: `var` declarations get set to `undefined`, function declarations get their full body stored, and `let`/`const` are registered but left uninitialized in the Temporal Dead Zone. Then in the execution phase, the code actually runs line by line. This two-phase creation is the mechanism behind hoisting.
>
> Contexts are managed on the call stack — pushed when a function is called, popped when it returns. The outer reference in the lexical environment is how closures work: a function defined inside another function retains a reference to the outer context's lexical environment, keeping it alive even after the outer function has returned."

---

## 14. Exercises

### Exercise 1 — Predict the output

```javascript
console.log(a);
console.log(b);
console.log(c);

var a = 1;
let b = 2;
const c = 3;
```

<details>
<summary>Answer</summary>

```
console.log(a) → undefined    (var hoisted, set to undefined in creation phase)
console.log(b) → ReferenceError: Cannot access 'b' before initialization (TDZ)
console.log(c) → never reached (error thrown on line 2)
```

</details>

---

### Exercise 2 — Trace the creation phase

Write out the complete Variable Environment and Lexical Environment after the creation phase of calling `run()`:

```javascript
var globalVar = "global";

function run(x) {
  var localVar = "local";
  let blockLet = "block";

  function helper() {
    return x;
  }

  return helper();
}

run(42);
```

<details>
<summary>Answer</summary>

```
Global EC — creation phase:
  Variable Env: { globalVar: undefined, run: ƒ run }
  Lexical Env:  { outer: null }

run() FEC — creation phase (when run(42) is called):
  Variable Env: { x: 42, localVar: undefined, helper: ƒ helper }
  Lexical Env:  { blockLet: <uninitialized>, outer: → Global LE }
  this:         window (default binding)
```

</details>

---

### Exercise 3 — Fix the `this` problem

```javascript
class Stopwatch {
  constructor() {
    this.elapsed = 0;
    this.intervalId = null;
  }

  start() {
    this.intervalId = setInterval(function () {
      this.elapsed++;
      console.log(this.elapsed);
    }, 1000);
  }

  stop() {
    clearInterval(this.intervalId);
  }
}

const sw = new Stopwatch();
sw.start(); // What happens? Fix it.
```

<details>
<summary>Answer</summary>

**What happens:** `this` inside the `setInterval` callback is `undefined` (strict mode) or `window` (sloppy mode). `this.elapsed++` either throws a TypeError or increments `window.elapsed`, not the Stopwatch instance.

**Fix:**

```javascript
start() {
  // Arrow function: this inherited from start()'s context = Stopwatch instance
  this.intervalId = setInterval(() => {
    this.elapsed++;
    console.log(this.elapsed);
  }, 1000);
}
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/02-call-stack.md`](./02-call-stack.md) — How contexts stack up
- [`javascript-core/05-closures.md`](./05-closures.md) — Lexical env kept alive
- [`javascript-core/07-scope-chain.md`](./07-scope-chain.md) — Outer references in depth
- [`javascript-core/03-event-loop.md`](./03-event-loop.md) — How contexts interact with async

---

<div align="center">

**Next:** [`02-call-stack.md`](./02-call-stack.md) →

</div>
