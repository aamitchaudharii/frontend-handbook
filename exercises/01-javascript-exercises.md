# 01 — JavaScript Exercises

> **"An exercise is a deliberate practice rep. The goal isn't to solve the problem — the goal is to build muscle memory for the underlying concept so you can apply it fluently in situations you haven't seen before."**

These exercises are grouped by concept, ordered from foundational to advanced within each section. Each includes a problem statement, hints, a reference solution, and a note on what concept it's designed to reinforce. Work through them without looking at the solutions first — the struggle is where the learning happens.

---

## Closures and Scope

### Exercise 1.1 — Make It Work

```javascript
// The following code should print 0, 1, 2, 3, 4 (one per second).
// It currently prints 5, 5, 5, 5, 5. Fix it — three different ways.

for (var i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i);
  }, i * 1000);
}
```

<details>
<summary>Solution</summary>

```javascript
// Fix 1: use let (block-scoped — new binding per iteration)
for (let i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i);
  }, i * 1000);
}

// Fix 2: IIFE to capture current value of i
for (var i = 0; i < 5; i++) {
  (function (j) {
    setTimeout(function () {
      console.log(j); // j is captured per IIFE call
    }, j * 1000);
  })(i);
}

// Fix 3: pass i as the third argument to setTimeout
// (setTimeout passes additional args to the callback)
for (var i = 0; i < 5; i++) {
  setTimeout(
    function (j) {
      console.log(j);
    },
    i * 1000,
    i,
  );
}

// Concept: closures capture VARIABLES, not values.
// All five var-based callbacks share the same `i` variable.
// By the time any callback runs, i is already 5.
// let creates a new binding per iteration → each closure captures a different variable.
```

</details>

---

### Exercise 1.2 — Implement `once`

```javascript
// Implement a function `once(fn)` that returns a wrapper which:
// - Calls fn on the first invocation and returns its result
// - Returns that same result on ALL subsequent calls without calling fn again

function once(fn) {
  // your implementation
}

// Test:
const initialize = once(() => {
  console.log("Initializing...");
  return 42;
});

console.log(initialize()); // logs "Initializing...", returns 42
console.log(initialize()); // returns 42 (no log)
console.log(initialize()); // returns 42 (no log)
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

// Concept: closures enable private state.
// `called` and `result` are captured by the returned function
// but inaccessible from the outside.
// Each call to once() creates a NEW closure with its OWN `called` and `result`.
```

</details>

---

### Exercise 1.3 — Implement `memoize`

```javascript
// Implement a memoize function that caches the results of a function call.
// Same arguments → return cached result without calling fn again.
// Arguments are primitives only (no need for deep equality).

function memoize(fn) {
  // your implementation
}

// Test:
let callCount = 0;
const expensiveSquare = memoize((n) => {
  callCount++;
  return n * n;
});

expensiveSquare(4); // 16, callCount = 1
expensiveSquare(4); // 16, callCount still 1 (cached)
expensiveSquare(5); // 25, callCount = 2
expensiveSquare(5); // 25, callCount still 2 (cached)
```

<details>
<summary>Solution</summary>

```javascript
function memoize(fn) {
  const cache = new Map();

  return function (...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Concept: closures + data structures.
// The cache Map lives in the closure, shared across all calls to the memoized function.
// JSON.stringify works as a key for primitive arguments.
// For complex arguments: you'd need a more sophisticated key function (e.g., a WeakMap).

// Extension: implement memoize with a max cache size (LRU eviction).
```

</details>

---

## Promises and Async

### Exercise 2.1 — Implement `promiseAll`

```javascript
// Implement Promise.all from scratch.
// Returns a promise that:
// - Resolves with an array of results when ALL promises resolve
// - Rejects immediately when ANY promise rejects

function promiseAll(promises) {
  // your implementation
}

// Test:
promiseAll([Promise.resolve(1), Promise.resolve(2), Promise.resolve(3)]).then(
  (values) => console.log(values),
); // [1, 2, 3]

promiseAll([
  Promise.resolve(1),
  Promise.reject(new Error("fail")),
  Promise.resolve(3),
]).catch((err) => console.log(err.message)); // "fail"
```

<details>
<summary>Solution</summary>

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      resolve([]);
      return;
    }

    const results = new Array(promises.length);
    let resolved = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise) // handle non-Promise values
        .then((value) => {
          results[index] = value; // preserve original order
          resolved++;
          if (resolved === promises.length) {
            resolve(results);
          }
        })
        .catch(reject); // any rejection → reject the whole thing
    });
  });
}

// Key insight: results must preserve ORDER, not arrival order.
// Use the index to store results in the correct position.
// Track resolved count: when it equals promises.length, all are done.
```

</details>

---

### Exercise 2.2 — Async Sequential vs Parallel

```javascript
// The following function fetches data for a list of user IDs.
// It's currently sequential (each fetch waits for the previous).
// Rewrite it to fetch ALL users in parallel.

async function fetchUsers(ids) {
  const users = [];
  for (const id of ids) {
    const user = await fetch(`/api/users/${id}`).then((r) => r.json());
    users.push(user);
  }
  return users;
}

// How does performance change? What's the tradeoff of parallel vs sequential?
```

<details>
<summary>Solution</summary>

```javascript
// Parallel fetch:
async function fetchUsers(ids) {
  return Promise.all(
    ids.map((id) => fetch(`/api/users/${id}`).then((r) => r.json())),
  );
}

// Performance:
// Sequential: total time = sum of all request times (N * avg_time)
// Parallel:   total time = max of all request times (1 * max_time)
// For 10 requests averaging 100ms: sequential = 1000ms, parallel = ~100ms

// Tradeoffs:
// Parallel: faster, but puts more concurrent load on the server
//           if one fails, Promise.all rejects immediately (fail-fast)
//           may hit rate limits for large ID lists

// For rate-limiting: process in batches
async function fetchUsersInBatches(ids, batchSize = 5) {
  const results = [];
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    const batchResults = await Promise.all(
      batch.map((id) => fetch(`/api/users/${id}`).then((r) => r.json())),
    );
    results.push(...batchResults);
  }
  return results;
}
```

</details>

---

### Exercise 2.3 — Implement `retry`

```javascript
// Implement a retry function that:
// - Calls an async function
// - If it fails, waits `delayMs` and tries again
// - Gives up after `maxRetries` attempts
// - Returns the result of the first success, or throws the last error

async function retry(fn, maxRetries = 3, delayMs = 1000) {
  // your implementation
}

// Test:
let attempts = 0;
const flaky = () => {
  attempts++;
  if (attempts < 3)
    return Promise.reject(new Error(`Attempt ${attempts} failed`));
  return Promise.resolve("success");
};

retry(flaky).then(console.log); // "success" (after 2 retries)
```

<details>
<summary>Solution</summary>

```javascript
async function retry(fn, maxRetries = 3, delayMs = 1000) {
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      if (attempt < maxRetries) {
        await new Promise((resolve) => setTimeout(resolve, delayMs));
      }
    }
  }

  throw lastError; // all attempts exhausted
}

// Extension: exponential backoff
async function retryWithBackoff(fn, maxRetries = 3, baseDelayMs = 1000) {
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      if (attempt < maxRetries) {
        const delay = Math.min(baseDelayMs * 2 ** (attempt - 1), 30_000);
        await new Promise((resolve) => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError;
}
```

</details>

---

## Functional Programming

### Exercise 3.1 — Implement `pipe` and `compose`

```javascript
// pipe: applies functions left-to-right
// compose: applies functions right-to-left (opposite order)

// pipe(f, g, h)(x) = h(g(f(x)))
// compose(f, g, h)(x) = f(g(h(x)))

function pipe(...fns) {
  // your implementation
}

function compose(...fns) {
  // your implementation
}

// Test:
const double = (x) => x * 2;
const addOne = (x) => x + 1;
const square = (x) => x * x;

pipe(double, addOne, square)(3); // square(addOne(double(3))) = square(7) = 49
compose(square, addOne, double)(3); // square(addOne(double(3))) = same = 49
```

<details>
<summary>Solution</summary>

```javascript
function pipe(...fns) {
  return (x) => fns.reduce((acc, fn) => fn(acc), x);
}

function compose(...fns) {
  return (x) => fns.reduceRight((acc, fn) => fn(acc), x);
}

// Concept: function composition — combining functions into a pipeline.
// reduce processes left-to-right (natural for pipe).
// reduceRight processes right-to-left (natural for compose, mathematical notation).

// Async pipe (handles promises):
function pipeAsync(...fns) {
  return (x) => fns.reduce((acc, fn) => Promise.resolve(acc).then(fn), x);
}
```

</details>

---

### Exercise 3.2 — Implement `flatMap`

```javascript
// Implement Array.prototype.flatMap from scratch.
// flatMap(fn) = flat(map(fn), 1 level)

Array.prototype.myFlatMap = function (fn) {
  // your implementation
};

// Test:
[1, 2, 3].myFlatMap((x) => [x, x * 2]); // [1, 2, 2, 4, 3, 6]
["hello world", "foo bar"].myFlatMap((s) => s.split(" ")); // ['hello', 'world', 'foo', 'bar']
```

<details>
<summary>Solution</summary>

```javascript
Array.prototype.myFlatMap = function (fn) {
  return this.reduce((acc, item, index, arr) => {
    const result = fn(item, index, arr);
    return Array.isArray(result) ? [...acc, ...result] : [...acc, result];
  }, []);
};

// Or more simply:
Array.prototype.myFlatMap = function (fn) {
  return [].concat(...this.map(fn));
};

// Concept: understanding reduce as a general-purpose aggregation.
// flatMap is equivalent to map + flat(1) — it maps and then flattens one level.
// Useful for: expanding items into variable-length sequences (as shown above).
```

</details>

---

## Data Structures and Algorithms

### Exercise 4.1 — Debounce

```javascript
// Implement debounce(fn, delay):
// Returns a function that delays invoking fn until `delay` ms after
// the last call. Useful for: search inputs, resize handlers.

function debounce(fn, delay) {
  // your implementation
}

// Test:
const searchApi = debounce((query) => {
  console.log(`Searching for: ${query}`);
}, 300);

searchApi("h"); // doesn't call immediately
searchApi("he"); // resets the timer
searchApi("hel"); // resets the timer
searchApi("hell"); // resets the timer
searchApi("hello"); // 300ms later: "Searching for: hello" (only once)
```

<details>
<summary>Solution</summary>

```javascript
function debounce(fn, delay) {
  let timerId = null;

  return function (...args) {
    clearTimeout(timerId); // cancel previous pending call
    timerId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

// Extension: debounce with leading option (fire immediately on first call)
function debounce(fn, delay, { leading = false } = {}) {
  let timerId = null;
  let hasBeenCalled = false;

  return function (...args) {
    if (leading && !hasBeenCalled) {
      fn.apply(this, args);
      hasBeenCalled = true;
    }

    clearTimeout(timerId);
    timerId = setTimeout(() => {
      if (!leading) fn.apply(this, args);
      hasBeenCalled = false;
    }, delay);
  };
}

// Concept: closures capture mutable state (timerId) that persists between calls.
// Each debounced function has its own timerId through closure.
```

</details>

---

### Exercise 4.2 — Throttle

```javascript
// Implement throttle(fn, interval):
// Returns a function that calls fn at most once per `interval` ms.
// Unlike debounce, throttle guarantees a call every interval (not just after silence).
// Useful for: scroll events, mouse move.

function throttle(fn, interval) {
  // your implementation
}

// Test: if called 10 times in 1 second with interval=200:
// Should fire at ~t=0, t=200, t=400, t=600, t=800, t=1000 (5-6 times total)
```

<details>
<summary>Solution</summary>

```javascript
// Leading-edge throttle: fires immediately, then ignores calls during cooldown
function throttle(fn, interval) {
  let lastCallTime = 0;

  return function (...args) {
    const now = Date.now();
    if (now - lastCallTime >= interval) {
      lastCallTime = now;
      fn.apply(this, args);
    }
  };
}

// Trailing-edge throttle: fires at end of interval (like setInterval with cleanup)
function throttleTrailing(fn, interval) {
  let timerId = null;
  let lastArgs = null;

  return function (...args) {
    lastArgs = args;
    if (!timerId) {
      timerId = setTimeout(() => {
        fn.apply(this, lastArgs);
        timerId = null;
      }, interval);
    }
  };
}

// Concept: throttle vs debounce.
// Throttle: regular calls at most every interval — good for continuous events (scroll)
// Debounce: waits for silence — good for completion events (search input stopped)
```

</details>

---

### Exercise 4.3 — Deep Clone

```javascript
// Implement deepClone(obj) without using JSON.parse/JSON.stringify.
// Handle: objects, arrays, primitives, Date, Map, Set, circular references.

function deepClone(obj, seen = new WeakMap()) {
  // your implementation
}

// Test:
const original = {
  name: "Alice",
  tags: ["admin", "user"],
  meta: { createdAt: new Date("2024-01-01"), score: 42 },
};
original.self = original; // circular reference

const clone = deepClone(original);
clone.tags.push("mod");
console.log(original.tags.length); // 2 (not affected)
console.log(clone.self === clone); // true (circular ref preserved)
```

<details>
<summary>Solution</summary>

```javascript
function deepClone(obj, seen = new WeakMap()) {
  // Primitives: return as-is
  if (obj === null || typeof obj !== "object") return obj;

  // Handle circular references
  if (seen.has(obj)) return seen.get(obj);

  // Date
  if (obj instanceof Date) return new Date(obj.getTime());

  // Array
  if (Array.isArray(obj)) {
    const clone = [];
    seen.set(obj, clone); // register BEFORE recursing (for circular refs)
    for (let i = 0; i < obj.length; i++) {
      clone[i] = deepClone(obj[i], seen);
    }
    return clone;
  }

  // Map
  if (obj instanceof Map) {
    const clone = new Map();
    seen.set(obj, clone);
    for (const [key, value] of obj) {
      clone.set(deepClone(key, seen), deepClone(value, seen));
    }
    return clone;
  }

  // Set
  if (obj instanceof Set) {
    const clone = new Set();
    seen.set(obj, clone);
    for (const value of obj) {
      clone.add(deepClone(value, seen));
    }
    return clone;
  }

  // Plain object
  const clone = Object.create(Object.getPrototypeOf(obj));
  seen.set(obj, clone); // register BEFORE recursing
  for (const key of Object.keys(obj)) {
    clone[key] = deepClone(obj[key], seen);
  }
  return clone;
}

// Key concepts:
// WeakMap tracks circular references — if we've seen this object, return the clone
// Must register clone BEFORE recursing into it (otherwise circular refs stack overflow)
// Modern alternative: structuredClone(obj) handles most cases natively
```

</details>

---

## Prototypes and OOP

### Exercise 5.1 — Implement `new` from scratch

```javascript
// Implement myNew(Constructor, ...args) that mimics the `new` keyword behavior.
// Steps that `new` performs:
// 1. Creates a new empty object
// 2. Sets the object's [[Prototype]] to Constructor.prototype
// 3. Calls the constructor with `this` = the new object
// 4. Returns the new object (or the constructor's return value if it's an object)

function myNew(Constructor, ...args) {
  // your implementation
}

// Test:
function Person(name, age) {
  this.name = name;
  this.age = age;
}
Person.prototype.greet = function () {
  return `Hi, I'm ${this.name}`;
};

const alice = myNew(Person, "Alice", 30);
alice.greet(); // "Hi, I'm Alice"
alice instanceof Person; // true
Object.getPrototypeOf(alice) === Person.prototype; // true
```

<details>
<summary>Solution</summary>

```javascript
function myNew(Constructor, ...args) {
  // 1. Create a new object with Constructor.prototype as its [[Prototype]]
  const obj = Object.create(Constructor.prototype);

  // 2. Call the constructor with the new object as `this`
  const result = Constructor.apply(obj, args);

  // 3. Return the constructor's return value IF it's an object,
  //    otherwise return the new object
  return result !== null && typeof result === "object" ? result : obj;
}

// Concept: understanding how `new` works under the hood.
// Object.create(Constructor.prototype) is the prototype chain link.
// Constructor.apply(obj, args) runs the constructor with `this` = obj.
// The return rule: constructors usually don't explicitly return — returning `obj` is the default.
// But if a constructor returns an object: that object is used instead of `this`.
```

</details>

---

### Exercise 5.2 — Implement `EventEmitter`

```javascript
// Implement a simple EventEmitter class with:
// on(event, listener): register a listener
// off(event, listener): remove a listener
// emit(event, ...args): call all listeners for this event
// once(event, listener): register a listener that fires only once

class EventEmitter {
  // your implementation
}

// Test:
const emitter = new EventEmitter();
const fn = (msg) => console.log(`Received: ${msg}`);

emitter.on("message", fn);
emitter.emit("message", "hello"); // "Received: hello"
emitter.emit("message", "world"); // "Received: world"
emitter.off("message", fn);
emitter.emit("message", "bye"); // nothing (listener removed)

emitter.once("connect", () => console.log("connected"));
emitter.emit("connect"); // "connected"
emitter.emit("connect"); // nothing (once listener removed)
```

<details>
<summary>Solution</summary>

```javascript
class EventEmitter {
  #listeners = new Map(); // event → Set of listeners

  on(event, listener) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(listener);
    return this; // enable chaining
  }

  off(event, listener) {
    this.#listeners.get(event)?.delete(listener);
    return this;
  }

  emit(event, ...args) {
    const listeners = this.#listeners.get(event);
    if (listeners) {
      // Snapshot the set before iterating — in case a listener calls off()
      [...listeners].forEach((listener) => listener(...args));
    }
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      listener(...args);
      this.off(event, wrapper); // remove WRAPPER, not original listener
    };
    return this.on(event, wrapper);
  }
}

// Concepts: Map of Sets for efficient lookup + Set for deduplication.
// once: wraps the listener in a self-removing function.
// Snapshot on emit: prevents issues if listeners modify the listener set during emit.
```

</details>

---

## Quick Practice Problems

```javascript
// PROBLEM 1: Flatten a nested array to any depth
// flatten([1, [2, [3, [4]], 5]]) → [1, 2, 3, 4, 5]
const flatten = (arr) =>
  arr.reduce(
    (acc, item) => acc.concat(Array.isArray(item) ? flatten(item) : item),
    [],
  );
// Modern: arr.flat(Infinity)

// PROBLEM 2: Group an array of objects by a key
// groupBy([{type:'a',v:1},{type:'b',v:2},{type:'a',v:3}], 'type')
// → { a: [{type:'a',v:1},{type:'a',v:3}], b: [{type:'b',v:2}] }
const groupBy = (arr, key) =>
  arr.reduce((acc, item) => {
    const k = item[key];
    return { ...acc, [k]: [...(acc[k] ?? []), item] };
  }, {});

// PROBLEM 3: Deep equal comparison
function deepEqual(a, b) {
  if (a === b) return true;
  if (typeof a !== typeof b) return false;
  if (typeof a !== "object" || a === null) return false;
  const keysA = Object.keys(a),
    keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;
  return keysA.every((key) => deepEqual(a[key], b[key]));
}

// PROBLEM 4: Chunk an array into groups of size n
// chunk([1,2,3,4,5], 2) → [[1,2],[3,4],[5]]
const chunk = (arr, n) =>
  Array.from({ length: Math.ceil(arr.length / n) }, (_, i) =>
    arr.slice(i * n, i * n + n),
  );
```

---

## 🔗 Related Topics

- [`javascript-core/04-closures.md`](../javascript-core/04-closures.md)
- [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md)
- [`javascript-core/11-promise-internals.md`](../javascript-core/11-promise-internals.md)
- [`exercises/02-react-exercises.md`](./02-react-exercises.md)
