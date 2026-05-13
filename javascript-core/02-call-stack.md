# 02 — The Call Stack

> **"The call stack is JavaScript's bookkeeping system. It remembers where you are, where you came from, and where to return — in every nested function call, across every execution context."**

The call stack is one of the most important data structures in the JavaScript runtime. Understanding it deeply means you can read stack traces fluently, predict recursion limits, debug async code, and understand _exactly_ why JavaScript is single-threaded. This document covers it from the ground up — mechanically, visually, and practically.

---

## 📚 Table of Contents

1. [What Is the Call Stack?](#1-what-is-the-call-stack)
2. [How the Call Stack Works — Step by Step](#2-how-the-call-stack-works--step-by-step)
3. [Stack Frames in Detail](#3-stack-frames-in-detail)
4. [Call Stack and Execution Contexts](#4-call-stack-and-execution-contexts)
5. [Stack Overflow — Causes and Mechanics](#5-stack-overflow--causes-and-mechanics)
6. [Reading Stack Traces](#6-reading-stack-traces)
7. [The Call Stack and Asynchronous Code](#7-the-call-stack-and-asynchronous-code)
8. [Call Stack in the Browser vs Node.js](#8-call-stack-in-the-browser-vs-nodejs)
9. [Tail Call Optimization](#9-tail-call-optimization)
10. [Debugging with the Call Stack](#10-debugging-with-the-call-stack)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. What Is the Call Stack?

The call stack is a **LIFO (Last In, First Out)** data structure that the JavaScript engine uses to track function execution. It records:

- Which function is currently running
- Which function called it (and where to return to)
- The execution context for each active function call

```
LIFO principle — the last function pushed is the first popped:

        PUSH (function called)     POP (function returns)
           │                            │
           ▼                            │
      ┌─────────┐                  ┌─────────┐
      │  inner  │ ← top (running)  │         │ ← inner popped
      ├─────────┤                  ├─────────┤
      │  outer  │                  │  outer  │ ← now running
      ├─────────┤                  ├─────────┤
      │ global  │                  │ global  │
      └─────────┘                  └─────────┘
```

**Key fact:** At any moment, the JavaScript engine is only executing the **topmost** frame on the call stack. Everything else is suspended, waiting to resume.

---

## 2. How the Call Stack Works — Step by Step

Let's trace a simple example in complete detail:

```javascript
function multiply(a, b) {
  return a * b;
}

function square(n) {
  return multiply(n, n);
}

function printSquare(n) {
  const result = square(n);
  console.log(result);
}

printSquare(4);
```

### Frame-by-Frame Trace

```
═══════════════════════════════════════
INITIAL STATE — script starts loading
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  main() / anonymous │  ← global script frame
└─────────────────────┘
Global EC created, code starts executing.

═══════════════════════════════════════
printSquare(4) CALLED
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  printSquare(4)     │  ← pushed. n=4. Paused at: const result = square(n)
├─────────────────────┤
│  main()             │
└─────────────────────┘

═══════════════════════════════════════
square(4) CALLED (from inside printSquare)
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  square(4)          │  ← pushed. n=4. Paused at: return multiply(n, n)
├─────────────────────┤
│  printSquare(4)     │  ← suspended, waiting for square to return
├─────────────────────┤
│  main()             │
└─────────────────────┘

═══════════════════════════════════════
multiply(4, 4) CALLED (from inside square)
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  multiply(4, 4)     │  ← pushed. a=4, b=4. Executing: return a * b
├─────────────────────┤
│  square(4)          │  ← suspended
├─────────────────────┤
│  printSquare(4)     │  ← suspended
├─────────────────────┤
│  main()             │
└─────────────────────┘

═══════════════════════════════════════
multiply RETURNS 16
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  square(4)          │  ← resumes. return value = 16
├─────────────────────┤
│  printSquare(4)     │  ← still suspended
├─────────────────────┤
│  main()             │
└─────────────────────┘
multiply frame POPPED. Return value 16 given to square.

═══════════════════════════════════════
square RETURNS 16
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  printSquare(4)     │  ← resumes. result = 16
├─────────────────────┤
│  main()             │
└─────────────────────┘
square frame POPPED. Return value 16 given to printSquare.

═══════════════════════════════════════
console.log(16) CALLED
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  console.log(16)    │  ← pushed (native call)
├─────────────────────┤
│  printSquare(4)     │  ← suspended
├─────────────────────┤
│  main()             │
└─────────────────────┘
Outputs: 16

═══════════════════════════════════════
console.log RETURNS, then printSquare RETURNS
═══════════════════════════════════════
Call Stack:
┌─────────────────────┐
│  main()             │  ← script ends
└─────────────────────┘
Then main() pops. Stack is empty.
Event loop can now run queued tasks.
```

---

## 3. Stack Frames in Detail

Each entry on the call stack is called a **stack frame** (or activation record). A frame contains:

```
Stack Frame for: multiply(4, 4)
┌─────────────────────────────────────────┐
│  Function reference: multiply           │
│  Arguments: a=4, b=4                    │
│  Local variables: (none in this fn)     │
│  Return address: line 6 in square()     │
│  Pointer to Execution Context           │
│  Pointer to outer scope (for closures)  │
└─────────────────────────────────────────┘
```

**Return address** is critical — it tells the engine exactly where to continue in the _calling_ function once the current function returns. Without this, nested calls would be impossible.

### Frame Size

Stack frames consume memory. The size depends on:

- Number of local variables
- Size of arguments
- Internal bookkeeping by the engine

V8 allocates a fixed stack size per thread (typically ~1MB in Chrome). Each frame adds to this. Too many nested frames = stack overflow.

---

## 4. Call Stack and Execution Contexts

The call stack and execution contexts are closely linked but distinct concepts:

```
Call Stack tracks:        Execution Context contains:
  Which function is         Variable bindings
  currently running         Lexical environment
  (the frame)               this binding
                            Scope chain references
```

```
Call Stack frame  ←──── points to ────►  Execution Context (in heap)
(lightweight,                            (heavier, contains all
 stack-allocated)                         variable bindings,
                                          may persist as closure)
```

When a frame is popped from the call stack, its execution context may or may not be garbage collected — it depends on whether any closures hold references to it.

```javascript
function outer() {
  const x = 10; // in outer's EC

  function inner() {
    return x; // inner closes over outer's EC
  }

  return inner;
}

const fn = outer();
// outer's stack frame: POPPED (gone)
// outer's execution context: STILL IN HEAP (fn closes over it)
fn(); // 10 — outer's EC still accessible via closure
```

---

## 5. Stack Overflow — Causes and Mechanics

A stack overflow occurs when the call stack **exceeds its size limit**. The engine throws:

```
RangeError: Maximum call stack size exceeded
```

### Cause 1 — Infinite Recursion (No Base Case)

```javascript
// ❌ No base case — recurses infinitely
function countdown(n) {
  console.log(n);
  countdown(n - 1); // always calls itself, never stops
}

countdown(10);
// Prints: 10, 9, 8, 7, 6, 5 ...
// Eventually: RangeError: Maximum call stack size exceeded
```

### Cause 2 — Mutual Recursion Without Termination

```javascript
// ❌ A calls B, B calls A, forever
function isEven(n) {
  if (n === 0) return true;
  return isOdd(n - 1);
}

function isOdd(n) {
  if (n === 0) return false;
  return isEven(n - 1);
}

isEven(100000); // Stack overflow on large values
```

### Cause 3 — Deep Call Trees

```javascript
// Not infinite — but deep enough to overflow the stack
function parseNestedJSON(obj, depth = 0) {
  // Deeply nested objects = deeply nested calls
  if (typeof obj !== "object") return obj;
  return Object.entries(obj).map(
    ([k, v]) => [k, parseNestedJSON(v, depth + 1)], // can overflow for depth > ~10,000
  );
}
```

### Stack Size Limits (Approximate)

| Environment             | Approximate call stack depth |
| ----------------------- | ---------------------------- |
| Chrome (V8)             | ~10,000–15,000 frames        |
| Firefox (SpiderMonkey)  | ~50,000 frames               |
| Node.js (V8)            | ~10,000–15,000 frames        |
| Safari (JavaScriptCore) | ~35,000–50,000 frames        |

These limits vary based on frame size — a function with many local variables has a larger frame, reducing the maximum depth.

### Fix: Convert Recursion to Iteration

```javascript
// ❌ Recursive — will overflow for large n
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// ✅ Iterative — constant stack depth
function fibonacci(n) {
  if (n <= 1) return n;
  let prev = 0,
    curr = 1;
  for (let i = 2; i <= n; i++) {
    [prev, curr] = [curr, prev + curr];
  }
  return curr;
}
```

### Fix: Trampolining for Recursive Algorithms

When recursion is genuinely the right structure, use a **trampoline** to convert deep recursion to a loop:

```javascript
// Trampoline: keeps calling functions that return functions
// until a non-function (the final result) is returned
function trampoline(fn) {
  return function (...args) {
    let result = fn(...args);
    while (typeof result === "function") {
      result = result(); // call the returned function
    }
    return result;
  };
}

// Recursive function rewritten to return a thunk instead of calling itself
function _factorial(n, acc = 1) {
  if (n <= 1) return acc;
  return () => _factorial(n - 1, n * acc); // return thunk instead of recurse
}

const factorial = trampoline(_factorial);
factorial(100000); // Works — never more than 1 frame deep at a time
```

---

## 6. Reading Stack Traces

When an error is thrown, the browser prints a **stack trace** — a snapshot of the call stack at the moment the error occurred. Reading it efficiently is a critical debugging skill.

### Anatomy of a Stack Trace

```
Error: Something went wrong
    at validateInput (utils.js:42:18)
    at processForm (form.js:87:5)
    at handleSubmit (handlers.js:23:3)
    at HTMLButtonElement.onclick (index.html:15:1)

FORMAT: at [functionName] ([file]:[line]:[column])
```

Read from **top to bottom**: the top is where the error was thrown. Each line below is the call that led to it.

```
onclick          ← user clicked button (outermost — started the chain)
  └── handleSubmit  ← called by onclick
        └── processForm   ← called by handleSubmit
              └── validateInput  ← ERROR THROWN HERE (innermost)
```

So the bug is in `validateInput` at line 42, but the chain of calls that led there goes through `handleSubmit → processForm → validateInput`.

### Real-World Stack Trace Example

```
TypeError: Cannot read properties of undefined (reading 'name')
    at UserCard.render (UserCard.js:34:22)
    at ComponentTree.update (tree.js:88:14)
    at BatchUpdater.flush (updater.js:56:8)
    at requestAnimationFrame (async)
    at BatchUpdater.schedule (updater.js:42:5)
    at UserCard.setState (Component.js:120:3)
    at UserCard.onDataLoaded (UserCard.js:28:10)
    at fetch.then (UserCard.js:20:16)

Reading this trace:
1. Error: undefined.name — something is undefined that shouldn't be
2. Error thrown in UserCard.render line 34 — that's where to look first
3. render was called by ComponentTree.update
4. Which was called by BatchUpdater.flush via requestAnimationFrame
5. The update was triggered by UserCard.setState
6. Which was called from onDataLoaded
7. Which was triggered by a fetch .then callback

Root cause: onDataLoaded likely called setState with incomplete data,
resulting in render receiving undefined where it expected a user object.
```

### Async Stack Traces (Modern Chrome)

Chrome DevTools shows async stack traces — the stack across `await` boundaries:

```
Error: Network failure
    at parseResponse (api.js:22:11)     ← synchronous part
    at async loadUser (api.js:15:18)    ← awaited function
    at async UserProfile.mount (profile.js:33:5)   ← awaited this
```

The `async` keyword in the trace shows where the stack crossed an `await` boundary.

---

## 7. The Call Stack and Asynchronous Code

This is a source of major confusion. Understanding it fully unlocks async debugging.

### The Key Rule

**Async callbacks are NOT on the call stack while they're waiting.** They're held by the browser's Web APIs. Only when their condition is met (timer fires, fetch completes, etc.) are they moved to the task queue — and only executed when the call stack is empty.

```javascript
console.log("A");

setTimeout(function timeout() {
  console.log("B");
}, 0);

console.log("C");
```

```
Timeline:

Call Stack:              Web APIs:           Task Queue:

[main]                   setTimeout(fn,0)     []
  → console.log('A')     ↓ (timer starts)
  → console.log: A

[main]                   setTimeout(fn,0)     []
  → setTimeout(fn,0)     (timer already
  → timer registered       near-instant)

[main]                   (timer fires)        [timeout fn]
  → console.log('C')     → fn moved to queue
  → console.log: C

[empty]                                       [timeout fn]
  ← stack is empty
  ← event loop checks queue

[timeout fn]             (empty)              []
  → console.log('B')
  → console.log: B

[empty]
```

Output: A, C, B

**The key insight:** `timeout fn` is NEVER on the call stack at the same time as `main`. The event loop only pushes it once the stack is clear.

### What "Blocking the Call Stack" Means

```javascript
// This prevents setTimeout callback from running for 5 seconds
// even though the timer fires immediately
setTimeout(() => console.log("timeout"), 0);

// This synchronous loop BLOCKS the call stack for 5 seconds
const start = Date.now();
while (Date.now() - start < 5000) {
  // busy wait
}

// The timeout callback can ONLY run after this loop finishes
// and the call stack empties
```

This is exactly why long synchronous operations freeze the UI — the call stack is never empty, so no callbacks (including rendering, user events, or timers) can run.

---

## 8. Call Stack in the Browser vs Node.js

Both use V8, but the host environment differs:

| Aspect               | Browser                   | Node.js                    |
| -------------------- | ------------------------- | -------------------------- |
| Initial global frame | `anonymous` / `(program)` | `Module` wrapper           |
| Global object        | `window`                  | `global` / `globalThis`    |
| Max stack depth      | ~10,000–15,000            | ~10,000–15,000             |
| Stack trace format   | Shows file URLs           | Shows file paths           |
| Async traces         | Yes (DevTools)            | Yes (--async-stack-traces) |

### Node.js Module Wrapper

Every Node.js file is wrapped in a function by the module system:

```javascript
// What Node.js actually runs:
(function (exports, require, module, __filename, __dirname) {
  // YOUR CODE HERE
  const x = 10;
  module.exports = x;
});

// So the stack trace in Node.js always shows this wrapper:
// at Object.<anonymous> (yourfile.js:1:1)
// That's the module wrapper function
```

---

## 9. Tail Call Optimization

**Tail Call Optimization (TCO)** is a compiler optimization where the engine reuses the current stack frame for a tail call — a function call that is the **last operation** in a function, with nothing left to do after it returns.

```javascript
// Non-tail call: must keep frame alive to add 1 after return
function notTailCall(n) {
  if (n === 0) return 0;
  return 1 + notTailCall(n - 1); // must wait for recursive result to add 1
  // Frame stays on stack waiting for the addition
}

// Tail call: nothing to do after the return — frame can be discarded
function tailCall(n, acc = 0) {
  if (n === 0) return acc;
  return tailCall(n - 1, acc + 1); // just returns the recursive result directly
  // Frame CAN be replaced (not stacked) — O(1) stack instead of O(n)
}
```

### TCO in JavaScript

TCO is part of the ES2015 spec but **only implemented in Safari (JavaScriptCore)**. Chrome and Node.js (V8) removed their experimental TCO implementations. So in practice:

```javascript
// This will still overflow in Chrome/Node.js despite being a proper tail call
function tailFactorial(n, acc = 1) {
  if (n <= 1) return acc;
  return tailFactorial(n - 1, n * acc); // proper tail call
}

tailFactorial(1000000); // RangeError in V8 — TCO not implemented
```

**Practical implication:** Don't rely on TCO in production JavaScript. Use the trampoline pattern or iterative implementations for deep recursion.

---

## 10. Debugging with the Call Stack

### Using DevTools Breakpoints

```
1. Open DevTools → Sources tab
2. Find your file → click line number to set breakpoint
3. Trigger the code
4. Execution pauses at the breakpoint
5. Call Stack panel (right side) shows the current stack
6. Click any frame to jump to that context and inspect its variables
7. Use Step Into (F11) to follow execution into called functions
8. Use Step Over (F10) to execute current line without entering sub-calls
9. Use Step Out (Shift+F11) to finish current function and return to caller
```

### Reading the DevTools Call Stack Panel

```
DevTools Call Stack panel during a paused breakpoint:

▶ handleClick        ← current (topmost) frame
  processInput       ← called handleClick
  EventListener.onclick  ← triggered the whole chain
  (anonymous)        ← global script
```

Clicking `processInput` in the panel:

- Jumps source view to `processInput`'s code
- Scope panel shows `processInput`'s local variables
- You're inspecting a suspended, mid-execution call

### `console.trace()` — Programmatic Stack Snapshots

```javascript
function deepUtilityFunction() {
  console.trace("Tracing call to deepUtilityFunction");
  // Prints the current call stack to the console
  // Useful for understanding what's calling a function
}
```

### Error Capturing for Stack Traces

```javascript
// Capture a stack trace without throwing an error
function captureStack() {
  const err = new Error();
  return err.stack;
}

// Or with Error.captureStackTrace (V8/Node.js):
function MyError(message) {
  this.message = message;
  Error.captureStackTrace(this, MyError); // excludes MyError from the trace
}
```

---

## 11. Good Practices

### ✅ Keep functions small — short, focused stack frames

```javascript
// ✅ Shallow, readable call stack
function validateAndSave(data) {
  const errors = validate(data);
  if (errors.length) throw new ValidationError(errors);
  return save(data);
}
```

### ✅ Use meaningful function names for readable stack traces

```javascript
// ❌ Anonymous functions produce unreadable stack traces
fetch("/api")
  .then(function (r) {
    return r.json();
  })
  .then(function (data) {
    processData(data);
  });

// Stack trace: at <anonymous>:3:5  ← useless

// ✅ Named functions produce useful stack traces
fetch("/api")
  .then(function parseResponse(r) {
    return r.json();
  })
  .then(function handleData(data) {
    processData(data);
  });

// Stack trace: at handleData (app.js:8:3)  ← useful!
```

### ✅ Guard against deep recursion

```javascript
// ✅ Depth-limited recursion
function deepCopy(obj, depth = 0, maxDepth = 100) {
  if (depth > maxDepth) throw new Error("Object too deeply nested");
  if (typeof obj !== "object" || obj === null) return obj;
  return Object.fromEntries(
    Object.entries(obj).map(([k, v]) => [k, deepCopy(v, depth + 1, maxDepth)]),
  );
}
```

### ✅ Use iterative solutions for potentially large input

```javascript
// ✅ Iterative depth-first traversal — constant stack depth
function findNode(tree, targetId) {
  const stack = [tree]; // our own explicit stack
  while (stack.length) {
    const node = stack.pop();
    if (node.id === targetId) return node;
    if (node.children) stack.push(...node.children);
  }
  return null;
}
```

---

## 12. Bad Practices

### ❌ Unbounded recursion

```javascript
// ❌ No depth limit on user-controlled input
function renderTemplate(template) {
  if (template.children) {
    template.children.forEach((child) => renderTemplate(child));
  }
}
// If template comes from user input with 20,000 nested levels: stack overflow
```

### ❌ Anonymous callbacks everywhere

```javascript
// ❌ Stack traces become useless
promise
  .then((r) => r.json()) // "at <anonymous>"
  .then((d) => transform(d)) // "at <anonymous>"
  .catch((e) => handleError(e)); // "at <anonymous>"
```

### ❌ Synchronous blocking on the main thread

```javascript
// ❌ Keeps call stack occupied — blocks ALL async callbacks
function syncHeavyWork() {
  for (let i = 0; i < 1_000_000_000; i++) {
    Math.sqrt(i); // pointless computation keeping stack busy
  }
}
```

### ❌ Circular function calls

```javascript
// ❌ Not infinite, but deeply confusing call graph
function A() {
  return B();
}
function B() {
  return C();
}
function C() {
  return A();
} // circular — infinite stack overflow
```

---

## 13. Common Mistakes

### Mistake 1 — Thinking async callbacks are on the stack while waiting

```javascript
setTimeout(() => {
  console.log("in timeout");
}, 1000);

// The arrow function is NOT on the call stack during the 1000ms wait.
// It's held by the browser's timer API.
// It's only placed on the stack after:
// 1. The 1000ms elapses
// 2. The call stack is empty
// 3. The event loop picks it from the queue
```

### Mistake 2 — Stack trace line numbers are from _before_ transpilation

```javascript
// Your source (TypeScript/ESNext):
// line 42: const result = processData(input);

// Transpiled output (what V8 executes):
// line 847: var result = processData(input);

// Stack trace shows line 847 — useless without source maps
// Fix: always configure source maps in your build tool
```

### Mistake 3 — Believing `new Error().stack` is free

```javascript
// Capturing a stack trace is expensive — V8 must walk the full call stack
// Don't do this in hot paths
function hotPath() {
  const stack = new Error().stack; // expensive! called millions of times
  log(stack);
}
```

### Mistake 4 — Long call chains hiding the real culprit

```javascript
// If your stack trace is 50 frames deep, the bug is usually
// in the top 1-3 frames — not in the middle of the chain.
// Start debugging from the TOP of the stack trace, not the bottom.
```

---

## 14. Interview-Level Explanation

> **"What is the call stack and how does it work in JavaScript?"**

**Strong answer:**

> "The call stack is a LIFO data structure the JavaScript engine uses to track function execution. Every time a function is called, a stack frame is pushed onto the stack containing the function's local variables, parameters, and a return address pointing back to the calling code. When the function returns, its frame is popped and execution resumes in the frame below it.
>
> JavaScript is single-threaded, which means the call stack has one frame executing at any moment — the topmost one. All other frames are suspended, waiting to resume when the functions above them return.
>
> This single-thread constraint is why long synchronous operations freeze the UI. The call stack is occupied for the duration of that operation, and no callbacks, no event handlers, no animation frames can run until the stack empties.
>
> When the stack exceeds its size limit — typically around 10,000 frames in V8 — it throws a 'Maximum call stack size exceeded' error. This happens with infinite recursion or deeply nested recursive calls. The fix is to convert deep recursion to iteration, or use the trampoline pattern.
>
> Async callbacks are never on the call stack while they're waiting. A setTimeout callback sits in the browser's timer system, and only moves to the task queue when the timer fires, and only executes when the call stack is completely empty. That's the bridge between the call stack and the event loop."

---

## 15. Exercises

### Exercise 1 — Trace the stack

Draw the call stack at each step for this code:

```javascript
function c() {
  console.log("c");
}

function b() {
  console.log("b");
  c();
}

function a() {
  console.log("a");
  b();
}

a();
```

<details>
<summary>Answer</summary>

```
Step 1: a() called
Stack: [global, a]

Step 2: inside a, console.log('a') → prints 'a'
Stack: [global, a, console.log]  → pops → [global, a]

Step 3: b() called from a
Stack: [global, a, b]

Step 4: console.log('b') → prints 'b'
Stack: [global, a, b, console.log] → pops → [global, a, b]

Step 5: c() called from b
Stack: [global, a, b, c]

Step 6: console.log('c') → prints 'c'
Stack: [global, a, b, c, console.log] → pops → [global, a, b, c]

Step 7: c returns
Stack: [global, a, b]

Step 8: b returns
Stack: [global, a]

Step 9: a returns
Stack: [global]

Output: a, b, c
```

</details>

---

### Exercise 2 — Fix the stack overflow

```javascript
// ❌ This will stack overflow for large arrays — fix it
function sumArray(arr, index = 0) {
  if (index === arr.length) return 0;
  return arr[index] + sumArray(arr, index + 1);
}

const bigArray = new Array(100000).fill(1);
sumArray(bigArray); // RangeError: Maximum call stack size exceeded
```

<details>
<summary>Solution</summary>

```javascript
// ✅ Option 1: Simple iteration
function sumArray(arr) {
  let total = 0;
  for (let i = 0; i < arr.length; i++) {
    total += arr[i];
  }
  return total;
}

// ✅ Option 2: Array reduce (iterative internally)
const sum = arr.reduce((acc, val) => acc + val, 0);

// ✅ Option 3: Trampoline (if recursion is required)
function _sumArray(arr, index = 0, acc = 0) {
  if (index === arr.length) return acc;
  return () => _sumArray(arr, index + 1, acc + arr[index]);
}

const sumArray = trampoline(_sumArray);
```

</details>

---

### Exercise 3 — Read the stack trace

```
Uncaught TypeError: Cannot read properties of null (reading 'querySelector')
    at initSidebar (sidebar.js:14:24)
    at setupLayout (layout.js:38:5)
    at DOMContentLoaded (app.js:5:3)
```

Answer these questions:

1. What went wrong?
2. Where did it happen?
3. What called the function where it happened?
4. What should you investigate first?

<details>
<summary>Answer</summary>

```
1. What went wrong?
   → Tried to call .querySelector() on null — an expected DOM element was null

2. Where did it happen?
   → sidebar.js, line 14, column 24 — inside the initSidebar function

3. What called it?
   → setupLayout called initSidebar (layout.js:38)
   → DOMContentLoaded handler called setupLayout (app.js:5)

4. What to investigate first?
   → Look at sidebar.js line 14 — what element is being accessed?
   → Likely: document.querySelector('#sidebar').querySelector(...)
     where #sidebar doesn't exist in the DOM yet, or at all.
   → Check if the element selector is correct.
   → Check if the element exists in the DOM at the time initSidebar runs.
   → The DOMContentLoaded handler suggests timing might be fine,
     so more likely the selector itself is wrong or the element is missing.
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/01-execution-context.md`](./01-execution-context.md) — What's inside each stack frame
- [`javascript-core/03-event-loop.md`](./03-event-loop.md) — How the call stack interacts with async queues
- [`javascript-core/05-closures.md`](./05-closures.md) — How stack frames relate to closures
- [`debugging/01-performance-tab.md`](../debugging/01-performance-tab.md) — Reading flame graphs (visual call stacks)

---

<div align="center">

**Next:** [`javascript-core/04-microtask-vs-macrotask.md`](./04-microtask-vs-macrotask.md) →

</div>
