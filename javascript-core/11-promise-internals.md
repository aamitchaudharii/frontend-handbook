# 11 — Promise Internals

> **"Most developers use Promises like a black box. Once you open the box — understand the state machine, the microtask queue integration, the chaining mechanics — async code stops being mysterious and starts being predictable."**

Promises are not magic. They're a specification (the Promises/A+ spec, adopted into ES2015) implemented as a state machine with very specific rules about when and how callbacks are called, how values propagate through chains, and how errors bubble. This document builds a working Promise implementation from scratch, explaining every design decision along the way.

---

## 📚 Table of Contents

1. [The Promise Specification](#1-the-promise-specification)
2. [Promise State Machine](#2-promise-state-machine)
3. [Building a Promise from Scratch](#3-building-a-promise-from-scratch)
4. [The Resolution Procedure](#4-the-resolution-procedure)
5. [Promise Chaining — How `.then()` Works](#5-promise-chaining--how-then-works)
6. [Error Propagation Mechanics](#6-error-propagation-mechanics)
7. [Microtask Integration](#7-microtask-integration)
8. [Promise.resolve and Promise.reject](#8-promiseresolve-and-promisereject)
9. [Static Methods Internals](#9-static-methods-internals)
10. [Async/Await Desugaring](#10-asyncawait-desugaring)
11. [The PromiseResolveThenableJob](#11-the-promiseresolvethenablejob)
12. [Unhandled Rejection Tracking](#12-unhandled-rejection-tracking)
13. [Memory Implications of Promises](#13-memory-implications-of-promises)
14. [Common Internal Gotchas](#14-common-internal-gotchas)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Promise Specification

Promises in JavaScript are governed by two overlapping specifications:

**Promises/A+** — the community specification that defines the core behavior:

- States (pending, fulfilled, rejected)
- The `.then()` method contract
- The resolution procedure

**ECMAScript Specification** — extends Promises/A+ with:

- `Promise.resolve()`, `Promise.reject()`
- `Promise.all()`, `Promise.allSettled()`, `Promise.race()`, `Promise.any()`
- Integration with async/await
- Microtask scheduling (via `PromiseJobs`)

### The Core Contract

```
A Promise MUST:
1. Be in one of three states: pending, fulfilled, or rejected
2. Once fulfilled or rejected, NEVER change state
3. Call .then() handlers asynchronously (never synchronously)
4. Call handlers in the order they were registered
5. Call handlers at most once
6. Propagate values and errors through chains
```

Rule 3 is critical and often surprising: even if a Promise is already resolved, its `.then()` handler runs asynchronously (as a microtask), never inline.

```javascript
const p = Promise.resolve(42); // already resolved

p.then((v) => console.log("A:", v)); // scheduled as microtask — NOT called yet

console.log("B"); // runs first

// Output: B, A: 42
// Even though p was ALREADY resolved, .then() fires asynchronously
```

---

## 2. Promise State Machine

A Promise is fundamentally a state machine with three states and two possible transitions:

```
                    resolve(value)
     ┌──────────────────────────────► FULFILLED
     │                                 (immutable)
  PENDING
     │
     └──────────────────────────────► REJECTED
                    reject(reason)    (immutable)

Rules:
  - Starts as PENDING
  - PENDING → FULFILLED: one-way, permanent
  - PENDING → REJECTED: one-way, permanent
  - FULFILLED → anything: IMPOSSIBLE
  - REJECTED → anything: IMPOSSIBLE
  - resolve() and reject() are no-ops if already settled
```

```javascript
// Demonstrating immutability of settled state
const p = new Promise((resolve, reject) => {
  resolve("first"); // settles the Promise as FULFILLED
  resolve("second"); // no-op — already settled
  reject("error"); // no-op — already settled
});

p.then((v) => console.log(v)); // 'first' — only the first resolve counts
```

### State Internal Representation

```javascript
// Conceptually, a Promise holds:
{
  _state: 'pending' | 'fulfilled' | 'rejected',
  _result: undefined,      // the fulfilled value or rejection reason
  _fulfillReactions: [],   // .then(onFulfilled) callbacks queued while pending
  _rejectReactions: [],    // .then(onRejected) / .catch() callbacks queued while pending
}
```

---

## 3. Building a Promise from Scratch

Let's build a working Promise implementation step by step. This is the best way to understand what's really happening under the hood.

### Step 1 — Basic Structure

```javascript
class MyPromise {
  #state   = 'pending';  // 'pending' | 'fulfilled' | 'rejected'
  #result  = undefined;  // fulfilled value or rejection reason
  #fulfillCallbacks = []; // handlers waiting for fulfillment
  #rejectCallbacks  = []; // handlers waiting for rejection

  constructor(executor) {
    // executor runs synchronously
    try {
      executor(
        (value) => this.#resolve(value),
        (reason) => this.#reject(reason)
      );
    } catch (err) {
      // If executor throws, reject the Promise
      this.#reject(err);
    }
  }
```

### Step 2 — Resolve and Reject

```javascript
  #resolve(value) {
    if (this.#state !== 'pending') return; // already settled — no-op

    // Special case: if resolving with a thenable (another Promise or thenable)
    // we must follow the resolution procedure
    if (value !== null && (typeof value === 'object' || typeof value === 'function')) {
      let then;
      try {
        then = value.then;
      } catch (err) {
        this.#reject(err);
        return;
      }

      if (typeof then === 'function') {
        // Resolving with a thenable — delegate to it
        // This is the PromiseResolveThenableJob
        queueMicrotask(() => {
          try {
            then.call(
              value,
              (v) => this.#resolve(v),    // when thenable fulfills
              (r) => this.#reject(r)      // when thenable rejects
            );
          } catch (err) {
            this.#reject(err);
          }
        });
        return;
      }
    }

    // Resolve with a plain value
    this.#state  = 'fulfilled';
    this.#result = value;
    this.#notifyFulfill();
  }

  #reject(reason) {
    if (this.#state !== 'pending') return; // already settled — no-op

    this.#state  = 'rejected';
    this.#result = reason;
    this.#notifyReject();
  }
```

### Step 3 — Notify Handlers

```javascript
  #notifyFulfill() {
    // Schedule each fulfill callback as a microtask
    // NEVER called synchronously — spec requirement
    this.#fulfillCallbacks.forEach(callback => {
      queueMicrotask(() => callback(this.#result));
    });
    this.#fulfillCallbacks = [];
  }

  #notifyReject() {
    this.#rejectCallbacks.forEach(callback => {
      queueMicrotask(() => callback(this.#result));
    });
    this.#rejectCallbacks = [];
  }
```

### Step 4 — The `.then()` Method

This is the heart of Promise chaining. `.then()` always returns a NEW Promise:

```javascript
  then(onFulfilled, onRejected) {
    // Normalize handlers — if not functions, pass through the value/reason
    const fulfill = typeof onFulfilled === 'function' ? onFulfilled : v => v;
    const reject  = typeof onRejected  === 'function' ? onRejected  : r => { throw r; };

    // .then() returns a NEW Promise
    return new MyPromise((resolve, nextReject) => {
      // Handler wrapper: run the handler, resolve/reject the next Promise
      const handleFulfill = (value) => {
        try {
          resolve(fulfill(value)); // resolve next with handler's return value
        } catch (err) {
          nextReject(err);         // if handler throws, reject next
        }
      };

      const handleReject = (reason) => {
        try {
          resolve(reject(reason)); // if rejection handler returns, FULFILL next
        } catch (err) {
          nextReject(err);         // if rejection handler throws, reject next
        }
      };

      if (this.#state === 'fulfilled') {
        // Already fulfilled — schedule microtask immediately
        queueMicrotask(() => handleFulfill(this.#result));
      } else if (this.#state === 'rejected') {
        // Already rejected — schedule microtask immediately
        queueMicrotask(() => handleReject(this.#result));
      } else {
        // Still pending — queue for later
        this.#fulfillCallbacks.push(handleFulfill);
        this.#rejectCallbacks.push(handleReject);
      }
    });
  }
```

### Step 5 — `.catch()` and `.finally()`

```javascript
  catch(onRejected) {
    return this.then(undefined, onRejected);
    // .catch() is just .then() with no fulfill handler
  }

  finally(onFinally) {
    return this.then(
      // On fulfill: run onFinally, then pass through original value
      value => MyPromise.resolve(onFinally()).then(() => value),
      // On reject: run onFinally, then re-throw original reason
      reason => MyPromise.resolve(onFinally()).then(() => { throw reason; })
    );
    // Key: finally passes through the original value/reason
    // (unless onFinally itself throws or returns a rejecting Promise)
  }
```

### Step 6 — Static Methods

```javascript
  static resolve(value) {
    // If value is already a MyPromise, return it as-is
    if (value instanceof MyPromise) return value;
    return new MyPromise(resolve => resolve(value));
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }

  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const results  = [];
      let   settled  = 0;
      const total    = promises.length;

      if (total === 0) { resolve([]); return; }

      promises.forEach((p, i) => {
        MyPromise.resolve(p).then(value => {
          results[i] = value;
          if (++settled === total) resolve(results);
        }).catch(reject); // first rejection triggers rejection of all
      });
    });
  }
}
```

---

## 4. The Resolution Procedure

The resolution procedure is the algorithm that runs when `resolve(value)` is called. It handles three cases:

### Case 1 — Resolving with `this` (the same Promise)

```javascript
// Spec: if resolving a Promise with itself → TypeError
const p = new Promise((resolve) => {
  resolve(p); // ← resolving p with p itself
});
// TypeError: Chaining cycle detected for promise
```

### Case 2 — Resolving with a Thenable

A "thenable" is any object or function with a `.then` method — not just native Promises.

```javascript
// Resolving with a thenable causes the Promise to adopt the thenable's state
const thenable = {
  then(resolve, reject) {
    resolve(42); // this thenable resolves with 42
  },
};

Promise.resolve(thenable).then((v) => console.log(v)); // 42
```

### Case 3 — Resolving with a Plain Value

```javascript
// Most common case — just sets state to fulfilled with the value
Promise.resolve(42).then((v) => console.log(v)); // 42
```

### Why This Matters

```javascript
// Returning a thenable from .then() makes the chain WAIT for it
Promise.resolve("start")
  .then((v) => {
    return fetch("/api/data"); // fetch returns a thenable (Promise)
    // The next .then() waits for fetch to complete
    // (not for this .then() handler to return)
  })
  .then((response) => response.json()) // receives the Response from fetch
  .then((data) => console.log(data));
```

---

## 5. Promise Chaining — How `.then()` Works

The key insight: `.then()` always returns a **new Promise** whose resolution depends on the handler's return value.

### The Chain Resolution Rules

```
Handler returns:        → Next Promise:
───────────────────────────────────────────────────────
Plain value (42)        → Fulfilled with 42
undefined (no return)   → Fulfilled with undefined
Another Promise         → Adopts that Promise's state
Throws an error         → Rejected with that error
```

```javascript
Promise.resolve(1)
  .then((n) => n + 1) // returns 2 → next fulfills with 2
  .then((n) => Promise.resolve(n * 10)) // returns Promise → next waits for it
  .then((n) => {
    throw new Error("fail"); // throws → next rejects
  })
  .then((n) => console.log("skip")) // skipped (rejected)
  .catch((err) => {
    console.log(err.message); // 'fail' — rejection caught here
    return "recovered"; // returns value → next fulfills (recovery!)
  })
  .then((v) => console.log(v)); // 'recovered' — chain continues after .catch
```

### Recovery After `.catch()`

```javascript
// .catch() returns a new Promise
// If the catch handler returns normally, the chain RESUMES as fulfilled

Promise.reject(new Error("initial"))
  .catch((err) => {
    console.log("caught:", err.message); // 'caught: initial'
    return "default value"; // return → chain resumes as fulfilled
  })
  .then((value) => console.log("value:", value)) // 'value: default value'
  .catch((err) => console.log("not reached"));
```

### Passing Through Handlers

```javascript
// If onFulfilled is not a function, the value passes through
Promise.resolve(42)
  .then(null) // no handler — value passes through
  .then(undefined) // no handler — value passes through
  .then((v) => console.log(v)); // 42

// If onRejected is not a function, the rejection passes through
Promise.reject(new Error("oops"))
  .then((v) => console.log(v)) // no rejection handler — passes through
  .catch((err) => console.log(err.message)); // 'oops'
```

---

## 6. Error Propagation Mechanics

### Errors Bubble Down the Chain

```javascript
// An unhandled rejection skips .then() handlers and falls to .catch()
Promise.resolve()
  .then(() => {
    throw new Error("A");
  }) // throws
  .then(() => console.log("skipped")) // skipped — previous rejected
  .then(() => console.log("skipped")) // skipped
  .catch((err) => console.log("caught:", err.message)) // 'caught: A'
  .then(() => console.log("chain continues")); // resumes here
```

### Multiple `.catch()` Handlers

```javascript
// Each .catch() handles rejections from the chain above it
Promise.resolve()
  .then(() => {
    throw new Error("step 1 error");
  })
  .catch((err) => {
    console.log("first catch:", err.message); // handles 'step 1 error'
    return "recovered after step 1"; // resume chain as fulfilled
  })
  .then(() => {
    throw new Error("step 2 error");
  })
  .catch((err) => {
    console.log("second catch:", err.message); // handles 'step 2 error'
  });
```

### Re-throwing in `.catch()`

```javascript
// .catch() can re-throw to pass error to next .catch()
Promise.reject(new Error("original"))
  .catch((err) => {
    if (err.message === "original") {
      throw new Error("transformed"); // re-throw with new error
    }
    return "handled";
  })
  .catch((err) => console.log(err.message)); // 'transformed'
```

### `.finally()` Does Not Swallow Errors

```javascript
// .finally() runs BUT passes through the original error
Promise.reject(new Error("original"))
  .finally(() => {
    console.log("cleanup ran"); // runs
    // returns undefined — does NOT affect the chain's state
  })
  .catch((err) => console.log(err.message)); // 'original' — passed through

// BUT: if .finally() throws, THAT error takes over
Promise.reject(new Error("original"))
  .finally(() => {
    throw new Error("finally threw"); // overrides original!
  })
  .catch((err) => console.log(err.message)); // 'finally threw'
```

---

## 7. Microtask Integration

One of the most important spec requirements: Promise handlers MUST be called asynchronously — as microtasks, never synchronously.

### Why Asynchronous Handlers?

```javascript
// If handlers ran synchronously, this would be unpredictable:
let x = 0;

Promise.resolve().then(() => {
  console.log("x =", x); // would always be 0 if sync
});

x = 1; // would this be visible to the handler?

// With async (microtask) handlers:
// x = 1 runs first (synchronous), THEN the handler runs
// Output: x = 1 (handler always sees the final sync state)
```

### The Microtask Queue Interaction

```javascript
console.log("1");

Promise.resolve()
  .then(() => {
    console.log("2"); // microtask
    return Promise.resolve("nested");
  })
  .then((v) => console.log("3:", v)); // this runs AFTER the inner promise resolves

Promise.resolve().then(() => console.log("4")); // separate chain

console.log("5");

// Execution trace:
// Sync: 1, 5
// Microtask queue: [handler for '2', handler for '4']
// Run '2' handler:
//   logs '2'
//   returns Promise.resolve('nested')
//   → PromiseResolveThenableJob queued: [handler for '4', PRTJ]
// Run '4' handler: logs '4'
// Run PRTJ: resolves inner promise → queues handler for '3'
// Run '3' handler: logs '3: nested'

// Output: 1, 5, 2, 4, 3: nested
```

This is why returning `Promise.resolve()` from `.then()` adds 2 extra microtask ticks — the `PromiseResolveThenableJob`.

---

## 8. Promise.resolve and Promise.reject

### `Promise.resolve()` — Identity for Promises

```javascript
// Special rule: Promise.resolve(aPromise) returns aPromise ITSELF
// (if it's a native Promise from the same realm)
const p = Promise.resolve(42);

Promise.resolve(p) === p; // true — same object returned!
// No wrapping — Promise.resolve is idempotent for native Promises

// BUT: for foreign thenables or Promises from different realms:
const foreign = { then: (resolve) => resolve(42) }; // thenable, not a Promise
const wrapped = Promise.resolve(foreign);
wrapped !== foreign; // true — wrapped in a native Promise
```

### `Promise.reject()` — Never Assimilated

```javascript
// Promise.reject ALWAYS creates a new rejected Promise
// Even if you pass it a Promise — that Promise becomes the REASON
const existingPromise = Promise.resolve(42);
const rejected = Promise.reject(existingPromise);

// rejected is a rejected Promise whose reason IS existingPromise
rejected.catch((reason) => {
  console.log(reason === existingPromise); // true — the Promise itself is the reason
});
```

---

## 9. Static Methods Internals

### `Promise.all` — Implementation

```javascript
// Our implementation from Section 3, with edge cases:
Promise.all([]); // → Promise<[]> — resolves immediately with empty array
Promise.all([1, 2, 3]); // → Promise<[1, 2, 3]> — wraps non-Promises via Promise.resolve
Promise.all([p1, Promise.reject("fail"), p3]);
// → Rejects immediately when p2 rejects
// → p1 and p3 continue running but their results are ignored
```

### `Promise.allSettled` — Full Implementation

```javascript
function myAllSettled(promises) {
  return new Promise((resolve) => {
    if (promises.length === 0) {
      resolve([]);
      return;
    }

    const results = new Array(promises.length);
    let settled = 0;

    promises.forEach((p, i) => {
      Promise.resolve(p)
        .then((value) => {
          results[i] = { status: "fulfilled", value };
        })
        .catch((reason) => {
          results[i] = { status: "rejected", reason };
        })
        .finally(() => {
          if (++settled === promises.length) resolve(results);
        });
    });
  });
}
```

### `Promise.race` — First Settler Wins

```javascript
function myRace(promises) {
  return new Promise((resolve, reject) => {
    for (const p of promises) {
      // Each promise calls resolve/reject on the outer Promise
      // But since a Promise can only settle once, only the first wins
      Promise.resolve(p).then(resolve, reject);
    }
    // Edge case: empty array → forever pending
    // (unlike Promise.all which resolves with [])
  });
}
```

### `Promise.any` — First Fulfillment

```javascript
function myAny(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      reject(new AggregateError([], "All promises were rejected"));
      return;
    }

    const errors = new Array(promises.length);
    let rejected = 0;

    promises.forEach((p, i) => {
      Promise.resolve(p)
        .then(resolve) // first fulfillment wins
        .catch((reason) => {
          errors[i] = reason;
          if (++rejected === promises.length) {
            // All rejected → reject with AggregateError
            reject(new AggregateError(errors, "All promises were rejected"));
          }
        });
    });
  });
}
```

---

## 10. Async/Await Desugaring

`async/await` is syntactic sugar. Understanding the desugaring makes the behavior predictable.

### Basic Desugaring

```javascript
// This async function:
async function example(url) {
  const response = await fetch(url);
  const data = await response.json();
  return data;
}

// Is equivalent to:
function example(url) {
  return fetch(url)
    .then((response) => response.json())
    .then((data) => data);
}
```

### Desugaring with Error Handling

```javascript
// async/await with try/catch:
async function example() {
  try {
    const data = await fetchData();
    return process(data);
  } catch (err) {
    return handleError(err);
  }
}

// Equivalent Promise chain:
function example() {
  return fetchData()
    .then((data) => process(data))
    .catch((err) => handleError(err));
}
```

### async Functions ALWAYS Return Promises

```javascript
// Even without any await or return:
async function empty() {}
empty() instanceof Promise; // true
empty().then((v) => console.log(v)); // undefined

// Returning a value:
async function getNumber() {
  return 42;
}
getNumber().then((v) => console.log(v)); // 42

// Throwing in an async function rejects the returned Promise:
async function throws() {
  throw new Error("oops");
}
throws().catch((err) => console.log(err.message)); // 'oops'
```

### The `await` Pause Point

```javascript
async function trace() {
  console.log("A"); // sync
  const v = await Promise.resolve(1);
  // ^ suspends here, returns to caller
  // Everything below is a microtask continuation
  console.log("B", v); // microtask
  const w = await Promise.resolve(2);
  // ^ suspends again
  console.log("C", w); // microtask
}

console.log("before");
trace();
console.log("after");

// Output: before, A, after, B 1, C 2
```

### await on Non-Promises

```javascript
// await on a non-thenable wraps it in Promise.resolve()
async function example() {
  const n = await 42; // same as await Promise.resolve(42)
  console.log(n); // 42
}
// This still creates a microtask — execution is still async after await
```

---

## 11. The PromiseResolveThenableJob

This is the most subtle behavior in the Promise specification, and the source of the "+2 microtask ticks" phenomenon.

### What It Is

When a `.then()` handler returns a thenable (another Promise), the spec requires a special job called `PromiseResolveThenableJob` to run. This job is itself a microtask, and it calls the thenable's `.then()` method — which schedules another microtask.

Net result: **returning a Promise from a `.then()` handler adds 2 extra microtask ticks** compared to returning a plain value.

### Demonstration

```javascript
// Chain A: returns plain value
Promise.resolve()
  .then(() => "value") // returns plain value
  .then((v) => console.log("A:", v)); // runs in 2 microtask ticks from start

// Chain B: returns Promise.resolve()
Promise.resolve()
  .then(() => Promise.resolve("value")) // returns a thenable
  .then((v) => console.log("B:", v)); // runs in 4 microtask ticks from start

// Both chains fire simultaneously at start
// Microtask queue progression:

// Tick 1: [A-handler, B-handler]
// Run A-handler: returns 'value' → queues A-then handler
// Queue: [B-handler, A-then]

// Run B-handler: returns Promise.resolve('value')
//   → PromiseResolveThenableJob queued
// Queue: [A-then, PRTJ]

// Run A-then: logs 'A: value'                    ← A resolves here (tick 3)
// Queue: [PRTJ]

// Run PRTJ: calls Promise.resolve('value').then(resolve)
//   → schedules another microtask to resolve B's outer promise
// Queue: [resolve-B-outer]

// Run resolve-B-outer: B's outer promise resolves with 'value'
//   → queues B-then handler
// Queue: [B-then]

// Run B-then: logs 'B: value'                    ← B resolves here (tick 5)

// Output: A: value, B: value
// A fires before B despite both chains being identical in structure
```

### Why This Matters

```javascript
// Classic interview question:
Promise.resolve()
  .then(() => {
    console.log("A");
    return Promise.resolve(); // +2 ticks
  })
  .then(() => console.log("B")); // tick 5

Promise.resolve()
  .then(() => console.log("C")) // tick 2
  .then(() => console.log("D")); // tick 3

// Output: A, C, D, B
// B comes AFTER D because returning Promise.resolve() adds 2 ticks
```

---

## 12. Unhandled Rejection Tracking

### How Browsers Track Unhandled Rejections

Browsers track rejected Promises that have no rejection handler. The tracking works like this:

```
1. Promise rejects
2. Browser starts a timer (end of current microtask checkpoint)
3. If a .catch() or rejection handler is attached before timer fires:
   → Promise is "handled" — no warning
4. If timer fires with no handler:
   → 'unhandledrejection' event fires on window
```

```javascript
// This is "handled" — .catch() added synchronously
const p = Promise.reject(new Error("oops"));
p.catch((err) => console.log("handled:", err.message));

// This is "handled" — .catch() added in microtask (before browser checks)
const q = Promise.reject(new Error("oops"));
queueMicrotask(() => q.catch((err) => console.log("handled:", err.message)));

// This is UNHANDLED — .catch() added in macrotask (after browser checks)
const r = Promise.reject(new Error("oops"));
setTimeout(() => r.catch((err) => console.log("too late")), 0);
// 'unhandledrejection' event fires
```

### Listening for Unhandled Rejections

```javascript
// Global handler — catch anything that slipped through
window.addEventListener("unhandledrejection", (event) => {
  // event.promise: the rejected Promise
  // event.reason:  the rejection reason
  console.error("Unhandled rejection:", event.reason);

  // Prevent default browser behavior (console.error in DevTools)
  event.preventDefault();

  // Report to error monitoring
  errorReporter.captureException(event.reason, {
    mechanism: "unhandledPromiseRejection",
  });
});

// Also handle rejections that are handled AFTER the fact:
window.addEventListener("rejectionhandled", (event) => {
  // A promise that was previously unhandled now has a handler
  console.log("Late-handled rejection:", event.reason);
});
```

---

## 13. Memory Implications of Promises

### Promises Hold References

A Promise holds references to all its `.then()` handlers until it settles. After settling, it holds a reference to its result value (for late-attached handlers).

```javascript
// This Promise keeps hugeData alive while pending
const p = new Promise((resolve) => {
  const hugeData = new Array(1_000_000);
  // If resolve is never called: hugeData stays alive (held by the executor closure)
  // and the Promise stays pending (but nobody awaits it → unreachable → GC'd)
  setTimeout(() => resolve(hugeData), 10_000);
});

// p holds a reference to the resolve/reject functions via the constructor
// resolve holds a reference to the Promise's internal state
// The Promise's internal state holds pending handlers
// → Long-lived pending Promises with large closures = memory pressure
```

### Never-Resolving Promises

```javascript
// ❌ This Promise is pending forever — and holds references forever
const pending = new Promise(() => {}); // resolve/reject never called

// However: if `pending` itself is unreachable from roots,
// the GC will collect it despite being pending
let p = new Promise(() => {});
p = null; // p is GC'd — no reference held
```

### Promise Chain Memory

```javascript
// A long chain holds references to intermediate Promises
// until the chain settles

const chain = fetch("/api")
  .then((r) => r.json()) // intermediate Promise held
  .then((data) => process(data)) // intermediate Promise held
  .then((result) => render(result)); // intermediate Promise held

// Once the final Promise settles, intermediate Promises become GC-eligible
// (as long as no other references to them exist)

// ❌ Storing intermediate Promises prevents GC
const p1 = fetch("/api");
const p2 = p1.then((r) => r.json());
const p3 = p2.then((data) => process(data));
// p1, p2, p3 all stay alive if these variables are in a long-lived scope
```

---

## 14. Common Internal Gotchas

### Gotcha 1 — Synchronous executor, asynchronous handler

```javascript
// The EXECUTOR runs synchronously
// The HANDLER always runs asynchronously

let resolved = false;

const p = new Promise((resolve) => {
  resolve(42); // synchronous — runs now
  resolved = true; // this runs BEFORE any .then() handler
});

p.then((v) => {
  console.log("resolved was:", resolved); // true — executor finished first
});

console.log("after new Promise, resolved =", resolved); // true
```

### Gotcha 2 — `.then()` on a rejected Promise without a rejection handler

```javascript
// If you don't handle the rejection, it passes through
// But the .then() promise is still a NEW rejected promise

const p = Promise.reject(new Error("oops"));

const p2 = p.then((v) => console.log("fulfilled:", v));
// p2 is rejected — but p2.catch() was not called → unhandled rejection!

p2.catch((err) => console.log("caught on p2:", err.message)); // 'oops'
```

### Gotcha 3 — Returning `undefined` vs no return

```javascript
// Both produce Promise<undefined>, but the intent differs

// Explicit undefined:
promise.then(() => undefined); // fulfills next with undefined

// No return (implicit undefined):
promise.then(() => {
  doSomething(); // no return — same as returning undefined
}); // fulfills next with undefined

// Returning void 0 is the same:
promise.then(() => void 0); // still fulfills with undefined
```

### Gotcha 4 — `.catch()` after the error is caught

```javascript
// .catch() RECOVERS the chain — subsequent .then() runs as fulfilled

Promise.reject("error")
  .catch((e) => "recovered") // returns 'recovered' → fulfills
  .then((v) => console.log(v)) // 'recovered' — NOT skipped
  .catch((e) => console.log("not reached"));
```

### Gotcha 5 — `async` function that forgets to `await`

```javascript
// ❌ Forgot to await — result is a Promise, not the value
async function getUser() {
  return fetch("/api/user").then((r) => r.json());
}

async function display() {
  const user = getUser(); // ← forgot await!
  console.log(user.name); // undefined — user is a Promise
}

// ✅
async function display() {
  const user = await getUser();
  console.log(user.name); // correct
}
```

---

## 15. Interview-Level Explanation

> **"How do Promises work internally? How does chaining work?"**

**Strong answer:**

> "A Promise is a state machine with three states: pending, fulfilled, and rejected. The state only transitions once — from pending to either fulfilled or rejected — and then it's immutable. Internally, it holds the result value or rejection reason, plus arrays of fulfill and reject callbacks that are waiting for settlement.
>
> When you call `.then(onFulfilled, onRejected)`, it always returns a new Promise. If the original Promise is already settled, the appropriate handler is scheduled as a microtask — never called synchronously, even if the Promise is already resolved. If the Promise is still pending, the handler is queued in the callback arrays and scheduled when the Promise eventually settles.
>
> The chaining behavior depends on what the handler returns. If it returns a plain value, the next Promise in the chain fulfills with that value. If it throws, the next Promise rejects. If it returns another thenable — another Promise or any object with a `.then` method — the chain pauses and waits for that thenable to settle, then adopts its state. This is handled by the PromiseResolveThenableJob, which is itself a microtask, which is why returning `Promise.resolve()` from a `.then()` handler adds two extra microtask ticks compared to returning a plain value.
>
> Error propagation works by passing rejections through the chain until a rejection handler is found. An unhandled rejection bubbles down through all `.then()` handlers that have no rejection handler, until it hits a `.catch()` or the end of the chain. `.catch()` returns a new Promise that fulfills if the catch handler returns normally, so the chain can recover. `.finally()` is a special case — it runs on both fulfillment and rejection, but passes the original value or error through unless the finally handler itself throws.
>
> The async/await syntax is desugared into promise chains by the JavaScript engine. Each `await` is a pause point where the async function suspends and schedules the continuation as a microtask."

---

## 16. Exercises

### Exercise 1 — Predict the output

```javascript
Promise.resolve(1)
  .then((v) => {
    console.log(v);
    return Promise.resolve(2);
  })
  .then((v) => console.log(v));

Promise.resolve(3).then((v) => console.log(v));
```

<details>
<summary>Answer</summary>

```
Output: 1, 3, 2

Microtask queue trace:
Initial: [handler(→1), handler(→3)]

Run handler(→1): logs 1, returns Promise.resolve(2)
  → PromiseResolveThenableJob queued
Queue: [handler(→3), PRTJ]

Run handler(→3): logs 3
Queue: [PRTJ]

Run PRTJ: resolves with 2, queues handler(→2)
Queue: [resolve-microtask]

Run resolve-microtask: outer promise resolves with 2
Queue: [handler(→2)]

Run handler(→2): logs 2
```

</details>

---

### Exercise 2 — Implement `Promise.race` and `Promise.any` from scratch

```javascript
function myRace(promises) {
  /* implement */
}
function myAny(promises) {
  /* implement */
}

// Test myRace:
myRace([
  new Promise((resolve) => setTimeout(() => resolve("slow"), 200)),
  new Promise((resolve) => setTimeout(() => resolve("fast"), 50)),
]).then(console.log); // 'fast'

// Test myAny:
myAny([
  Promise.reject("err1"),
  new Promise((resolve) => setTimeout(() => resolve("success"), 100)),
  Promise.reject("err2"),
]).then(console.log); // 'success'

myAny([Promise.reject("err1"), Promise.reject("err2")]).catch((err) =>
  console.log(err instanceof AggregateError),
); // true
```

<details>
<summary>Solution</summary>

```javascript
function myRace(promises) {
  return new Promise((resolve, reject) => {
    for (const p of promises) {
      Promise.resolve(p).then(resolve, reject);
    }
    // Empty array: forever pending (matches spec)
  });
}

function myAny(promises) {
  return new Promise((resolve, reject) => {
    const arr = Array.from(promises);

    if (arr.length === 0) {
      reject(new AggregateError([], "All promises were rejected"));
      return;
    }

    const errors = new Array(arr.length);
    let rejected = 0;

    arr.forEach((p, i) => {
      Promise.resolve(p)
        .then(resolve) // first success wins
        .catch((reason) => {
          errors[i] = reason;
          if (++rejected === arr.length) {
            reject(new AggregateError(errors, "All promises were rejected"));
          }
        });
    });
  });
}
```

</details>

---

### Exercise 3 — Build a complete Promise from scratch

Using the skeleton from Section 3, complete the implementation so all of these tests pass:

```javascript
// Test 1: Basic resolution
new MyPromise((resolve) => resolve(42)).then((v) =>
  console.assert(v === 42, "Test 1"),
);

// Test 2: Rejection and catch
new MyPromise((_, reject) => reject(new Error("oops"))).catch((err) =>
  console.assert(err.message === "oops", "Test 2"),
);

// Test 3: Chaining
MyPromise.resolve(1)
  .then((v) => v + 1)
  .then((v) => v * 10)
  .then((v) => console.assert(v === 20, "Test 3"));

// Test 4: Async resolution
new MyPromise((resolve) => setTimeout(() => resolve("async"), 10)).then((v) =>
  console.assert(v === "async", "Test 4"),
);

// Test 5: Error recovery
MyPromise.reject(new Error("fail"))
  .catch(() => "recovered")
  .then((v) => console.assert(v === "recovered", "Test 5"));

// Test 6: Returning a Promise from .then()
MyPromise.resolve(1)
  .then((v) => MyPromise.resolve(v + 1))
  .then((v) => console.assert(v === 2, "Test 6"));

// Test 7: Finally
let finallyCalled = false;
MyPromise.resolve(42)
  .finally(() => {
    finallyCalled = true;
  })
  .then((v) => {
    console.assert(v === 42, "Test 7a — value passed through");
    console.assert(finallyCalled, "Test 7b — finally called");
  });
```

---

## 🔗 Related Topics

- [`javascript-core/03-event-loop.md`](./03-event-loop.md) — Microtask queue and the event loop
- [`javascript-core/04-microtask-vs-macrotask.md`](./04-microtask-vs-macrotask.md) — When Promise handlers run
- [`javascript-core/10-async-patterns.md`](./10-async-patterns.md) — Practical async patterns
- [`javascript-core/12-web-workers.md`](./12-web-workers.md) — True parallelism beyond async

---

<div align="center">

**Next:** [`javascript-core/12-web-workers.md`](./12-web-workers.md) →

</div>
