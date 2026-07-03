# 05 — Closures

> **"A closure is not a trick, a pattern, or a feature you choose to use. It's an automatic consequence of how JavaScript's scope system works. Every function you write is a closure."**

Closures are one of the most misunderstood concepts in JavaScript — not because they're complex, but because most explanations start with the definition rather than the mechanism. This document builds closures from the ground up: from lexical scope, through execution contexts, to the heap-resident environment records that make closures possible — and then covers every real-world pattern and failure mode.

---

## 📚 Table of Contents

1. [The Mechanism — What Actually Happens](#1-the-mechanism--what-actually-happens)
2. [Lexical Scope — The Foundation](#2-lexical-scope--the-foundation)
3. [How Closures Form](#3-how-closures-form)
4. [What a Closure Captures](#4-what-a-closure-captures)
5. [Closures and the Heap](#5-closures-and-the-heap)
6. [The Classic Loop Bug](#6-the-classic-loop-bug)
7. [Practical Closure Patterns](#7-practical-closure-patterns)
8. [Closures and Memory](#8-closures-and-memory)
9. [Closures in Modern JavaScript](#9-closures-in-modern-javascript)
10. [Performance Considerations](#10-performance-considerations)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. The Mechanism — What Actually Happens

Before any definition, let's see the mechanism:

```javascript
function outer() {
  let count = 0; // lives in outer's lexical environment

  return function inner() {
    count++; // references outer's environment
    return count;
  };
}

const increment = outer(); // outer() returns inner, outer's EC is popped
increment(); // 1          // but count is still accessible
increment(); // 2
increment(); // 3
```

When `outer()` returns, its execution context is removed from the call stack. In most languages, local variables are destroyed at this point. In JavaScript, they aren't — because `inner` holds a reference to `outer`'s **lexical environment**, which keeps it alive in the heap.

```
After outer() returns:

Call Stack: [empty — outer's frame is gone]

Heap:
  ┌─────────────────────────────────┐
  │  outer's Lexical Environment    │
  │  { count: 0 }                   │◄── referenced by inner
  └─────────────────────────────────┘

  ┌─────────────────────────────────┐
  │  inner function object          │
  │  [[Environment]] ──────────────►│ outer's Lexical Env
  └─────────────────────────────────┘
  ▲
  │ referenced by `increment`
```

The environment record (the actual storage of `count`) lives in the heap, not the stack. The stack frame is gone; the data is not.

---

## 2. Lexical Scope — The Foundation

To understand closures you must first understand **lexical scope**: scope is determined by where code is **written** in the source, not where it is **called** from.

```javascript
const x = "global";

function outer() {
  const x = "outer";

  function inner() {
    return x; // which x?
  }

  return inner;
}

const fn = outer();
fn(); // 'outer' — not 'global'
```

`inner` is defined **inside** `outer`, so it lexically belongs to `outer`'s scope. When `inner` looks up `x`, it finds `outer`'s `x` — regardless of where `inner` is later called from.

### Lexical vs Dynamic Scope

```
Lexical scope (JavaScript):
  Scope chain determined at WRITE TIME (where the code is written)
  inner's outer scope = wherever inner was defined

Dynamic scope (some other languages, NOT JavaScript):
  Scope chain determined at CALL TIME (where the code is called from)
  inner's outer scope = whoever called inner
```

```javascript
const x = "global";

function a() {
  const x = "inside a";
  b(); // calls b — but b is defined at global level
}

function b() {
  return x; // lexical: looks at where b was DEFINED, not called
}

a(); // 'global' (not 'inside a') — JavaScript is lexically scoped
```

---

## 3. How Closures Form

A closure forms whenever a function is defined inside another function and accesses variables from the outer function's scope. This happens automatically — you don't opt in to closures.

### The Three Ingredients

```
1. An outer function with local variables
2. An inner function that references those variables
3. The inner function outlives the outer function
   (returned, stored, passed as callback, etc.)
```

```javascript
// Ingredient 1: outer function with local variable
function makeMultiplier(factor) {
  // Ingredient 2: inner function references `factor`
  return function multiply(number) {
    return number * factor; // closes over `factor`
  };
  // Ingredient 3: inner function is returned (outlives outer)
}

const double = makeMultiplier(2);
const triple = makeMultiplier(3);

double(5); // 10 — factor = 2, still accessible
triple(5); // 15 — factor = 3, separate closure
```

Each call to `makeMultiplier` creates a **new, independent closure** — each `multiply` function has its own reference to its own `factor`.

### Closures Form on Definition, Not on Call

```javascript
function createFunctions() {
  const fns = [];

  for (var i = 0; i < 3; i++) {
    fns.push(function () {
      return i; // closes over i at DEFINITION TIME
    });
  }

  return fns;
}

const [f0, f1, f2] = createFunctions();
f0(); // 3 — not 0!
f1(); // 3 — not 1!
f2(); // 3 — not 2!
```

This is the classic closure loop bug (covered in detail in Section 6). All three functions close over the **same** `i` — because `var` creates one `i` for the entire function scope.

---

## 4. What a Closure Captures

This is the most important nuance: closures don't capture **values** — they capture **references** to **variable bindings**.

```javascript
function makeCounter() {
  let count = 0;

  return {
    increment: () => ++count, // same `count` binding
    decrement: () => --count, // same `count` binding
    get: () => count, // same `count` binding
  };
}

const counter = makeCounter();
counter.increment(); // count = 1
counter.increment(); // count = 2
counter.decrement(); // count = 1
counter.get(); // 1
```

All three returned functions close over the **same `count` variable**. When one modifies it, all see the change. There is one `count` in memory, and three functions referencing it.

### Mutable Shared State Through Closures

```javascript
function makeSharedState() {
  let state = { value: 0 };

  return {
    set: (v) => {
      state.value = v;
    },
    get: () => state.value,
    // Returning state directly would share the OBJECT reference
    // (but replacing `state` itself in set would break the sharing)
  };
}
```

### Closures Capture the Entire Environment Record

A closure doesn't just capture the specific variables it uses — it captures a reference to the entire environment record. V8 optimizes this (context variable analysis can determine which variables are actually needed and only keep those), but conceptually:

```javascript
function outer() {
  const huge = new Array(1_000_000).fill(0); // 8MB
  const small = 42;

  return function inner() {
    return small; // only uses `small`, not `huge`
    // But the entire outer environment record is technically referenced
    // Modern V8 optimizes this away — `huge` CAN be collected
    // But this is engine-dependent
  };
}
```

**Practical implication:** Don't create closures inside functions that allocate large data unnecessarily. See Section 8 (Closures and Memory).

---

## 5. Closures and the Heap

When a closure forms, the engine must decide: this variable can't live on the stack (the stack frame will be destroyed). It must move to the heap.

```
Without closure (variable on stack):
  outer() called → stack frame allocated → count=0 on stack
  outer() returns → stack frame freed → count destroyed ✓

With closure (variable on heap):
  outer() called → stack frame allocated
  inner defined → references outer's environment
  → V8: "count will be needed after outer returns — allocate on heap"
  outer() returns → stack frame freed
  → count still exists on heap, referenced by inner's [[Environment]]
  → count lives until inner is garbage collected
```

This heap allocation is what makes closures possible — and what makes them potentially expensive if misused.

### Environment Sharing Between Closures

Multiple closures defined in the same scope share the same environment record:

```javascript
function makeStore() {
  let data = {}; // shared environment record
  let version = 0; // same environment record

  // All three functions share ONE environment record
  return {
    set(key, val) {
      data[key] = val;
      version++;
    },
    get(key) {
      return data[key];
    },
    getVersion() {
      return version;
    },
  };
}

// One set of heap-allocated variables, three closures pointing to them
const store = makeStore();
store.set("a", 1);
store.get("a"); // 1
store.getVersion(); // 1
```

---

## 6. The Classic Loop Bug

The loop closure bug is arguably the most common JavaScript mistake related to closures. Understanding it deeply requires understanding the difference between `var` and `let` scoping.

### The Bug

```javascript
// ❌ Classic bug with var
const buttons = document.querySelectorAll("button");

for (var i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", function () {
    console.log(`Button ${i} clicked`);
  });
}

// Click any button → always logs "Button 5 clicked" (or whatever length is)
// Why? var is function-scoped — ONE `i` for the entire loop
// All 5 closures reference the SAME `i`
// By the time any button is clicked, the loop is done and i = 5
```

### Why It Happens — The Mental Model

```
With var:
  One variable `i` in the function scope

  Loop iteration 1: i = 0
    → closure created for button[0]
    → closure references the ONE `i` variable (currently 0)

  Loop iteration 2: i = 1
    → closure created for button[1]
    → closure references the SAME ONE `i` variable (now 1)

  ... loop ends ...

  i = 5 (or buttons.length)

  User clicks button[0]:
    → closure runs: console.log(`Button ${i}`) → i is 5 → "Button 5"
  User clicks button[1]:
    → same closure-referenced i → still 5 → "Button 5"
```

### Fix 1 — Use `let` (best solution)

```javascript
// ✅ let is block-scoped — a NEW binding per iteration
for (let i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", function () {
    console.log(`Button ${i} clicked`);
  });
}
// Each iteration has its own `i` binding
// Click button[0] → "Button 0", button[1] → "Button 1", etc.
```

With `let`, each loop iteration creates a new `i` variable in a new block scope. Each closure captures a different binding.

### Fix 2 — IIFE (historical, pre-ES6)

```javascript
// ✅ IIFE creates a new scope per iteration, capturing current i value
for (var i = 0; i < buttons.length; i++) {
  (function (capturedI) {
    buttons[capturedI].addEventListener("click", function () {
      console.log(`Button ${capturedI} clicked`);
    });
  })(i); // immediately invoke with current i, creating a new binding
}
```

### Fix 3 — Store in local variable

```javascript
// ✅ Capture value in a new local binding
for (var i = 0; i < buttons.length; i++) {
  const index = i; // new `const` binding per iteration = new closure scope
  buttons[index].addEventListener("click", function () {
    console.log(`Button ${index} clicked`);
  });
}
```

### Fix 4 — Use `forEach` (different scope per callback)

```javascript
// ✅ forEach callback creates a new scope for each element
Array.from(buttons).forEach((button, index) => {
  button.addEventListener("click", () => {
    console.log(`Button ${index} clicked`);
  });
});
```

---

## 7. Practical Closure Patterns

### Pattern 1 — Module Pattern (Private State)

The most important real-world use of closures. Creates encapsulation without classes:

```javascript
const UserService = (function () {
  // Private — not accessible outside
  let _users = [];
  let _nextId = 1;
  const _cache = new Map();

  // Private helper
  function _validate(user) {
    return user.name && user.email;
  }

  // Public API — closures over private state
  return {
    addUser(userData) {
      if (!_validate(userData)) throw new Error("Invalid user");
      const user = { ...userData, id: _nextId++ };
      _users.push(user);
      _cache.delete("all"); // invalidate cache
      return user;
    },

    getUsers() {
      if (!_cache.has("all")) {
        _cache.set("all", [..._users]); // copy to prevent mutation
      }
      return _cache.get("all");
    },

    getUserById(id) {
      return _users.find((u) => u.id === id);
    },

    get count() {
      return _users.length;
    },
  };
})();

UserService.addUser({ name: "Alice", email: "alice@example.com" });
UserService._users; // undefined — private, not accessible
UserService.count; // 1
```

### Pattern 2 — Function Factory

Create specialized functions from a general template:

```javascript
function makeValidator(rules) {
  // rules is captured in the closure
  return function validate(value) {
    const errors = [];
    for (const [ruleName, ruleCheck] of Object.entries(rules)) {
      if (!ruleCheck(value)) {
        errors.push(ruleName);
      }
    }
    return { valid: errors.length === 0, errors };
  };
}

const validateEmail = makeValidator({
  required: (v) => v && v.length > 0,
  hasAt: (v) => v.includes("@"),
  hasDomain: (v) => v.split("@")[1]?.includes("."),
  maxLength: (v) => v.length <= 254,
});

const validateAge = makeValidator({
  required: (v) => v !== null && v !== undefined,
  isNumber: (v) => typeof v === "number",
  isPositive: (v) => v > 0,
  isAdult: (v) => v >= 18,
});

validateEmail("alice@example.com"); // { valid: true, errors: [] }
validateAge(15); // { valid: false, errors: ['isAdult'] }
```

### Pattern 3 — Memoization

Cache expensive computation results using a closure:

```javascript
function memoize(fn) {
  const cache = new Map(); // closed over — persists between calls

  return function memoized(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const expensiveCalc = memoize(function (n) {
  console.log(`Computing ${n}...`);
  let result = 0;
  for (let i = 0; i <= n; i++) result += i;
  return result;
});

expensiveCalc(1000); // "Computing 1000..." → 500500
expensiveCalc(1000); // (cached, no log)    → 500500
expensiveCalc(2000); // "Computing 2000..." → 2001000
```

### Pattern 4 — Partial Application and Currying

```javascript
// Partial application — fix some arguments, return function for rest
function partial(fn, ...presetArgs) {
  return function (...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

function add(a, b, c) {
  return a + b + c;
}

const add5 = partial(add, 5); // fixes a=5
const add5and3 = partial(add, 5, 3); // fixes a=5, b=3

add5(3, 2); // 10 (5 + 3 + 2)
add5and3(10); // 18 (5 + 3 + 10)

// Currying — transform f(a,b,c) into f(a)(b)(c)
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function (...moreArgs) {
      return curried.apply(this, args.concat(moreArgs));
    };
  };
}

const curriedAdd = curry((a, b, c) => a + b + c);
curriedAdd(1)(2)(3); // 6
curriedAdd(1, 2)(3); // 6
curriedAdd(1)(2, 3); // 6
```

### Pattern 5 — Once Function

```javascript
// A function that can only be called once
function once(fn) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
  };
}

const initializeApp = once(() => {
  console.log("App initialized");
  return { status: "ready" };
});

initializeApp(); // "App initialized" → { status: 'ready' }
initializeApp(); // (nothing) → { status: 'ready' } (same result)
initializeApp(); // (nothing) → { status: 'ready' }
```

### Pattern 6 — Debounce and Throttle

```javascript
// Debounce: wait for N ms of inactivity, then call once
function debounce(fn, delay) {
  let timerId; // closed over — persists between calls

  return function debounced(...args) {
    clearTimeout(timerId); // cancel previous scheduled call
    timerId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

// Throttle: call at most once per N ms
function throttle(fn, limit) {
  let lastCall = 0; // closed over

  return function throttled(...args) {
    const now = Date.now();
    if (now - lastCall >= limit) {
      lastCall = now;
      return fn.apply(this, args);
    }
  };
}

const onSearch = debounce((query) => fetchResults(query), 300);
const onScroll = throttle((e) => updateScrollUI(e), 16);

searchInput.addEventListener("input", (e) => onSearch(e.target.value));
window.addEventListener("scroll", onScroll);
```

### Pattern 7 — Event Emitter

```javascript
function createEventEmitter() {
  const listeners = new Map(); // private, closed over

  return {
    on(event, fn) {
      if (!listeners.has(event)) listeners.set(event, new Set());
      listeners.get(event).add(fn);
      return () => listeners.get(event)?.delete(fn); // return unsubscribe
    },

    off(event, fn) {
      listeners.get(event)?.delete(fn);
    },

    emit(event, ...args) {
      listeners.get(event)?.forEach((fn) => fn(...args));
    },

    once(event, fn) {
      const unsub = this.on(event, (...args) => {
        fn(...args);
        unsub(); // auto-remove after first call
      });
      return unsub;
    },
  };
}

const emitter = createEventEmitter();
const unsub = emitter.on("data", console.log);
emitter.emit("data", "hello"); // "hello"
unsub(); // remove listener
emitter.emit("data", "hello"); // (nothing)
```

---

## 8. Closures and Memory

Closures keep their entire enclosing scope alive. This is their power — and their danger.

### What Gets Kept Alive

```javascript
function processLargeData() {
  const hugeArray = new Array(1_000_000).fill(0); // 8MB
  const config = { threshold: 100 };

  return function checkThreshold(value) {
    // Only uses `config`, not `hugeArray`
    // But hugeArray is in the same environment record
    // Whether V8 keeps it depends on its optimization level
    return value > config.threshold;
  };
}

const checker = processLargeData();
// `hugeArray` may or may not be GC'd — depends on V8's context analysis
```

### Closure Memory Leak Pattern

```javascript
// ❌ Common leak: closure keeps large data alive unintentionally
class DataProcessor {
  constructor(data) {
    this.data = data; // large dataset

    // ❌ This event listener closes over `this` (which has this.data)
    // Listener will keep entire DataProcessor alive as long as it's registered
    document.addEventListener("keydown", (e) => {
      if (e.key === "Escape") this.cancel(); // `this` = DataProcessor
    });
  }
}

// Each time a DataProcessor is created and "destroyed":
// listener still registered → DataProcessor not GC'd → data not freed
```

```javascript
// ✅ Fixed: store minimal reference, clean up on destroy
class DataProcessor {
  constructor(data) {
    this.data = data;

    // Only capture what's needed
    this._onKeydown = (e) => {
      if (e.key === "Escape") this.cancel();
    };
    document.addEventListener("keydown", this._onKeydown);
  }

  destroy() {
    document.removeEventListener("keydown", this._onKeydown);
    this._onKeydown = null;
    this.data = null; // release reference
  }
}
```

### Extracting Only What's Needed

```javascript
// ❌ Closes over entire `config` object (may be large)
function setupWithConfig(config) {
  return function process(value) {
    return value * config.multiplier + config.offset;
  };
}

// ✅ Extract only needed values — config can be GC'd after setup
function setupWithConfig(config) {
  const { multiplier, offset } = config; // extract primitives

  return function process(value) {
    return value * multiplier + offset; // captures only two numbers
  };
}
```

---

## 9. Closures in Modern JavaScript

### Class Private Fields vs Closure Privacy

ES2022 introduced true private class fields (`#field`). How do they compare to closure-based privacy?

```javascript
// Closure-based private state
function makeCounter(initial = 0) {
  let _count = initial; // genuinely private — no way to access from outside
  return {
    increment() {
      return ++_count;
    },
    get value() {
      return _count;
    },
  };
}

// Class with private fields
class Counter {
  #count; // private field — cannot be accessed outside class

  constructor(initial = 0) {
    this.#count = initial;
  }
  increment() {
    return ++this.#count;
  }
  get value() {
    return this.#count;
  }
}

// Both achieve private state
// Class version is more idiomatic for OOP patterns
// Closure version is more flexible (no `this`, easier to compose)
```

### Arrow Functions and Closures

Arrow functions are closures that also lexically bind `this`:

```javascript
class Component {
  constructor() {
    this.state = { count: 0 };
    this.element = document.createElement("div");

    // Arrow function: closes over `this` lexically
    // this = Component instance, always
    this.element.addEventListener("click", () => {
      this.state.count++; // this is correctly the Component
      this.render();
    });
  }
}
```

### Closures in React Hooks (Understanding the Pattern)

React hooks are built entirely on closures. Understanding this helps debug stale closure issues:

```javascript
// React useState under the hood (simplified concept)
function useState(initial) {
  let state = initial; // closed over

  function getState() {
    return state;
  }
  function setState(newVal) {
    state = newVal;
    rerender(); // trigger re-render
  }

  return [getState, setState];
}

// The "stale closure" problem (common React bug):
function Counter() {
  const [count, setCount] = useState(0);

  // ❌ Stale closure: this effect captures count=0 from the first render
  // When count changes via setCount, this callback still sees count=0
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(count + 1); // count is always 0 here (stale)
    }, 1000);
    return () => clearInterval(timer);
  }, []); // empty deps — effect runs once, captures initial count

  // ✅ Fix: use functional update form (doesn't need to close over count)
  useEffect(() => {
    const timer = setInterval(() => {
      setCount((c) => c + 1); // c is always current value — no closure needed
    }, 1000);
    return () => clearInterval(timer);
  }, []);
}
```

---

## 10. Performance Considerations

### Closure Creation Cost

Creating a closure allocates a new function object and potentially a new environment record on the heap. In hot paths, this matters:

```javascript
// ❌ Creates a new closure object on every render/call
function List({ items }) {
  return items.map((item) => (
    // New function created per item per render
    <Button onClick={() => handleClick(item.id)} />
  ));
}

// ✅ Stable reference — no new closure per render
function List({ items }) {
  const handleClick = useCallback((id) => {
    // handle click
  }, []); // created once

  return items.map((item) => (
    <Button onClick={() => handleClick(item.id)} />
    // Still a closure, but handleClick is stable
  ));
}
```

### V8 Context Analysis Optimization

V8 performs **context variable analysis** to minimize what's kept alive in closure environments. If a variable is never referenced in any inner function, it won't be kept in the closure environment:

```javascript
function optimized() {
  const used = 42;
  const unused = new Array(1_000_000).fill(0); // V8 may not include in closure

  return function inner() {
    return used; // only references `used`
  };
}
// V8 can determine `unused` is never referenced in inner
// and may exclude it from the captured environment
// (implementation detail — don't rely on this for memory safety)
```

### Avoid Closures in Tight Loops

```javascript
// ❌ Creates N closure objects in a tight loop
function processItems(items) {
  return items.map((item) => () => transform(item)); // N closures
}

// ✅ Create the function once, pass data differently
function transformItem(item) {
  return transform(item);
}
function processItems(items) {
  return items.map(transformItem); // one function reference, no closures
}
```

---

## 11. Good Practices

### ✅ Use closures for genuine encapsulation

```javascript
// ✅ Private state that truly can't be accessed or modified externally
function createRouter() {
  const routes = new Map(); // private
  let currentPath = "/"; // private

  return {
    register(path, handler) {
      routes.set(path, handler);
    },
    navigate(path) {
      currentPath = path;
      routes.get(path)?.();
    },
    get currentPath() {
      return currentPath;
    },
  };
}
```

### ✅ Return unsubscribe/cleanup functions from closures

```javascript
// ✅ Every closure that sets up a side effect returns its cleanup
function setupPolling(url, interval) {
  let timerId = setInterval(() => fetch(url), interval);

  // Return cleanup function — closure over timerId
  return function stop() {
    clearInterval(timerId);
    timerId = null;
  };
}

const stopPolling = setupPolling("/api/status", 5000);
// Later:
stopPolling(); // cleans up the interval
```

### ✅ Use `let`/`const` in loops, never `var`

```javascript
// ✅ Each iteration gets its own binding
for (let i = 0; i < 10; i++) {
  setTimeout(() => console.log(i), i * 100); // 0, 1, 2, 3...
}
```

### ✅ Extract large objects before creating a closure

```javascript
// ✅ Don't capture more than needed
function createHandler(bigConfig) {
  const endpoint = bigConfig.api.endpoint; // extract primitive
  const timeout = bigConfig.api.timeout;

  return async function handle(data) {
    return fetch(endpoint, { method: "POST", body: JSON.stringify(data) });
  };
  // bigConfig can be GC'd after createHandler returns
}
```

---

## 12. Bad Practices

### ❌ `var` in loops with async callbacks

```javascript
// ❌ All callbacks see the final value of i
for (var i = 0; i < arr.length; i++) {
  fetch(`/api/${arr[i]}`).then(() => console.log(i)); // always arr.length
}
```

### ❌ Closures that accidentally capture large scope

```javascript
// ❌ `data` is huge, closure keeps it alive
function setup(data) {
  const summary = summarize(data); // small result

  return function display() {
    showSummary(summary);
    // Doesn't use `data` but `data` is in the same scope
    // May prevent `data` from being GC'd
  };
}

// ✅ Isolate in a nested scope
function setup(data) {
  const summary = summarize(data);
  data = null; // explicitly release if not needed by other closures

  return function display() {
    showSummary(summary);
  };
}
```

### ❌ Mutating closed-over state unexpectedly

```javascript
// ❌ Surprising mutation through shared closure
function makePair() {
  let value = 0;
  return {
    getter: () => value,
    setter: (v) => {
      value = v;
    },
  };
}

const pair = makePair();
pair.setter(42);
pair.getter(); // 42 — this is intentional

// But if multiple parts of code receive the same pair object
// unexpected mutations can be hard to trace
```

### ❌ Deeply nested closures

```javascript
// ❌ Hard to reason about, deep scope chains
function a() {
  return function b() {
    return function c() {
      return function d() {
        // Scope chain: d → c → b → a → global
        // Hard to track which variables come from where
      };
    };
  };
}
```

---

## 13. Common Mistakes

### Mistake 1 — Thinking closures copy values

```javascript
let x = 1;

const fn = () => console.log(x); // closes over binding, not value

fn(); // 1
x = 99;
fn(); // 99 — closure sees updated binding, not original value
```

Closures capture the **binding** (the variable itself), not a snapshot of its value at the time the closure was created.

### Mistake 2 — The Loop Bug with `var`

Already covered in Section 6, but worth emphasizing: `var` creates ONE binding for the entire function. `let` creates a new binding per iteration.

### Mistake 3 — Arrow function as object method (loses `this`)

```javascript
const obj = {
  value: 42,
  // ❌ Arrow function: `this` = outer lexical context (probably window)
  getValue: () => this.value, // undefined or error
  // ✅ Regular method: `this` = obj when called as obj.getValue()
  getValueCorrect() {
    return this.value;
  },
};
```

### Mistake 4 — Stale closures in async code

```javascript
let count = 0;

async function update() {
  const snapshot = count; // reads count right now

  await delay(1000); // count may change during this wait

  // ❌ snapshot is stale if count changed during the await
  console.log("count was:", snapshot, "but is now:", count);
}

// Fix: read count AFTER the await if you need the current value
async function update() {
  await delay(1000);
  console.log("current count:", count); // always current
}
```

---

## 14. Interview-Level Explanation

> **"What is a closure in JavaScript?"**

**Strong answer:**

> "A closure is the combination of a function and the lexical environment in which it was defined. In practice: when a function is defined inside another function and accesses variables from the outer function's scope, those variables aren't destroyed when the outer function returns — they're kept alive in the heap because the inner function holds a reference to their environment record.
>
> The key mechanism is JavaScript's lexical scoping: scope is determined by where code is written, not where it's called from. When V8 creates a function, it attaches a `[[Environment]]` reference to the function object pointing to the lexical environment where it was defined. If that environment contains variables needed by the function, those variables are allocated on the heap rather than the stack, so they outlive the enclosing function call.
>
> Closures don't capture values — they capture variable bindings. If you change a variable after a closure is created, the closure sees the new value. This is why the classic `var` loop bug happens: all closures share one `var` binding that ends at the final loop value.
>
> In practice, closures power: the module pattern for private state, memoization by capturing a cache between calls, debounce/throttle by capturing timer IDs, and factory functions that produce specialized versions of a general function.
>
> The memory implication is important: a closure keeps its entire captured environment alive. A small callback that closes over a large dataset prevents that dataset from being garbage collected. This is a common source of memory leaks in SPAs where components create listeners but don't clean them up on unmount."

---

## 15. Exercises

### Exercise 1 — Predict the output

```javascript
function makeAdder(x) {
  return function (y) {
    return x + y;
  };
}

const add5 = makeAdder(5);
const add10 = makeAdder(10);

console.log(add5(3)); // ?
console.log(add10(3)); // ?
console.log(add5(add10(2))); // ?
```

<details>
<summary>Answer</summary>

```
add5(3)         → 5 + 3 = 8
add10(3)        → 10 + 3 = 13
add5(add10(2))  → add10(2) = 12, add5(12) = 17
```

</details>

---

### Exercise 2 — Fix the loop bug

```javascript
// Fix this so each timeout logs the correct index
for (var i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i);
  }, i * 1000);
}
// Currently logs: 5, 5, 5, 5, 5
// Should log: 0, 1, 2, 3, 4
```

<details>
<summary>Three solutions</summary>

```javascript
// Solution 1: let (best)
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), i * 1000);
}

// Solution 2: IIFE
for (var i = 0; i < 5; i++) {
  (function (j) {
    setTimeout(() => console.log(j), j * 1000);
  })(i);
}

// Solution 3: bind
for (var i = 0; i < 5; i++) {
  setTimeout(console.log.bind(console, i), i * 1000);
}
```

</details>

---

### Exercise 3 — Build a rate limiter

Implement a `createRateLimiter(fn, maxCalls, windowMs)` function that:

- Allows at most `maxCalls` invocations of `fn` within any `windowMs` window
- Returns the result of `fn` if within the limit
- Returns `null` and logs a warning if the limit is exceeded

Use closures to track call history.

<details>
<summary>Solution</summary>

```javascript
function createRateLimiter(fn, maxCalls, windowMs) {
  const callTimestamps = []; // closed over — persists between calls

  return function rateLimited(...args) {
    const now = Date.now();

    // Remove timestamps outside the current window
    while (callTimestamps.length > 0 && callTimestamps[0] < now - windowMs) {
      callTimestamps.shift();
    }

    if (callTimestamps.length >= maxCalls) {
      console.warn(
        `Rate limit exceeded: max ${maxCalls} calls per ${windowMs}ms`,
      );
      return null;
    }

    callTimestamps.push(now);
    return fn.apply(this, args);
  };
}

// Usage:
const limitedFetch = createRateLimiter(fetch, 5, 1000); // 5 calls per second
limitedFetch("/api/data"); // OK
limitedFetch("/api/data"); // OK
limitedFetch("/api/data"); // OK
limitedFetch("/api/data"); // OK
limitedFetch("/api/data"); // OK
limitedFetch("/api/data"); // "Rate limit exceeded" → null
```

</details>

---

### Exercise 4 — Implement `memoize` with cache expiry

Extend the memoize pattern to support TTL (time-to-live) for cached entries.

<details>
<summary>Solution</summary>

```javascript
function memoizeWithTTL(fn, ttl = 60000) {
  const cache = new Map(); // { key → { value, expiresAt } }

  return function memoized(...args) {
    const key = JSON.stringify(args);
    const now = Date.now();

    if (cache.has(key)) {
      const entry = cache.get(key);
      if (now < entry.expiresAt) {
        return entry.value; // cache hit, not expired
      }
      cache.delete(key); // expired — remove
    }

    const result = fn.apply(this, args);
    cache.set(key, { value: result, expiresAt: now + ttl });
    return result;
  };
}

const cachedFetch = memoizeWithTTL(
  (url) => fetch(url).then((r) => r.json()),
  5 * 60 * 1000, // 5 minute TTL
);
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/01-execution-context.md`](./01-execution-context.md) — The execution context that closures reference
- [`javascript-core/07-scope-chain.md`](./07-scope-chain.md) — How scope chain traversal works
- [`javascript-core/08-memory-management.md`](./08-memory-management.md) — Memory implications of closures
- [`performance/05-memory-leaks.md`](../performance/05-memory-leaks.md) — Closure-based memory leaks
- [`patterns/01-observer.md`](../patterns/01-observer.md) — Observer pattern using closures

---

<div align="center">

**Next:** [`javascript-core/06-prototypes.md`](./06-prototypes.md) →

</div>
