# 🧠 Mental Models

> **"A mental model is an internal representation of how something works. The right mental model makes debugging obvious. The wrong one makes everything a mystery."**

Before diving into the deep technical content in this handbook, you need the right mental models. These are not summaries of topics — they are the **frameworks for thinking** that senior engineers have internalized. With them, you can predict behavior you've never seen before. Without them, you memorize facts that don't connect.

This document covers the twelve most important mental models for frontend engineering. Each one is a lens you'll use repeatedly across every topic in this handbook.

---

## 📚 Table of Contents

1. [The Single-Thread Mental Model](#1-the-single-thread-mental-model)
2. [The Reachability Mental Model](#2-the-reachability-mental-model)
3. [The Pipeline Mental Model](#3-the-pipeline-mental-model)
4. [The Two-Phase Mental Model](#4-the-two-phase-mental-model)
5. [The Lexical vs Dynamic Mental Model](#5-the-lexical-vs-dynamic-mental-model)
6. [The Batching Mental Model](#6-the-batching-mental-model)
7. [The Delegation Mental Model](#7-the-delegation-mental-model)
8. [The Cost Model](#8-the-cost-model)
9. [The Tradeoff Mental Model](#9-the-tradeoff-mental-model)
10. [The Ownership Mental Model](#10-the-ownership-mental-model)
11. [The Isolation Mental Model](#11-the-isolation-mental-model)
12. [The Scale Mental Model](#12-the-scale-mental-model)

---

## 1. The Single-Thread Mental Model

**The model:** JavaScript runs on a single thread. Only one thing executes at a time. Everything else waits.

### Why This Model Matters

Most developers know JavaScript is single-threaded. Few have truly internalized what it means:

```
WRONG mental model (common):
  "My code runs. Meanwhile, fetch() runs in parallel. setTimeout fires
   while my code is still going. It's all happening at the same time."

RIGHT mental model:
  "My code is the only thing running right now. When fetch() is called,
   the browser starts a request on a SEPARATE networking thread.
   My code continues. When the network thread gets a response,
   it puts a callback in a queue. My code must FINISH before that
   callback can run. Everything happens one at a time."
```

### The Mental Picture

```
Main Thread Timeline:

──────────────────────────────────────────────────────────────────────▶ time
│  Script  │  Event  │  Script  │  Event  │ [render] │  Script  │
│ executes │ handler │ executes │ handler │          │ executes │

Only ONE box runs at a time.
The browser cannot render during Script execution.
User input cannot be processed during Script execution.
```

### When to Apply This Model

Apply it whenever you see:

- `setTimeout(fn, 0)` — why it doesn't run "immediately"
- UI freezing — your code is blocking the thread
- Animations stuttering — your JavaScript is eating the frame budget
- Async functions — why `await` suspends but doesn't block

### The Implication

```
The single-thread model explains why:
  ✦ A 500ms synchronous loop freezes the page for 500ms
  ✦ requestAnimationFrame runs BEFORE paint, not during it
  ✦ Promise.then() never runs while synchronous code is executing
  ✦ Web Workers exist — they're the ONLY way to truly parallelize JS
```

`→ Deep dive:` [`javascript-core/03-event-loop.md`](../javascript-core/03-event-loop.md)

---

## 2. The Reachability Mental Model

**The model:** Memory is kept alive by references, not by usage. Something is alive if it can be reached. It can be collected if it cannot.

### Why This Model Matters

```
WRONG mental model:
  "When a component unmounts, its memory is freed."
  "When a function returns, its variables are cleaned up."
  "When I reassign a variable, the old value is deleted."

RIGHT mental model:
  "Memory is freed ONLY when no path exists from any root
   (global, call stack, event listener) to the object.
   If even ONE reference remains — anywhere — the object stays alive."
```

### The Reference Graph

Think of memory as a directed graph:

```
GC Roots:
  window ──► app ──► userService ──► cache ──► [userData1]
                                              ──► [userData2]
               └──► eventBus ──► [handler] ──► [component] ──► [bigData]

Everything reachable via any path from a root: ALIVE
Everything NOT reachable: ELIGIBLE FOR COLLECTION
```

A "memory leak" is not broken GC. It is a **reference you forgot to remove**.

### The Three Leak Patterns

```
1. GLOBAL REFERENCE:
   window.cache = data;  // root → cache → data
   // data lives forever unless cache is cleared

2. CALLBACK REFERENCE:
   element.addEventListener('click', () => {
     doSomethingWith(component); // element → handler → component
   });
   // component lives as long as element exists

3. CIRCULAR WITH ROOT PATH:
   a → b → a  (circular)
   ↑
   root
   // Both a and b live forever — they're reachable from root
```

### When to Apply This Model

Apply it whenever:

- A component "should be gone" but the app slows down over time
- You're using `WeakMap`/`WeakRef` — the weakness is about reachability
- You add event listeners, timers, or subscriptions
- You cache data in module scope

`→ Deep dive:` [`javascript-core/08-memory-management.md`](../javascript-core/08-memory-management.md)

---

## 3. The Pipeline Mental Model

**The model:** Every rendered frame is the output of a sequential pipeline. Each stage transforms the output of the previous stage. Changing something early in the pipeline is more expensive than changing something late.

### The Pipeline

```
HTML bytes
   ↓ [parse]
DOM Tree
   ↓ [parse CSS, merge]
Render Tree (with computed styles)
   ↓ [layout engine]
Box Model Geometry (positions, sizes)
   ↓ [rasterizer]
Painted Pixels (per layer)
   ↓ [GPU compositor]
Screen
```

### The Stage Cost Table

```
Changing...          Triggers...         Cost
─────────────────────────────────────────────────
width, height        Layout+Paint+Comp   ████████ EXPENSIVE
display, position    Layout+Paint+Comp   ████████ EXPENSIVE
color, background    Paint+Comp          █████    MODERATE
box-shadow           Paint+Comp          ██████   MODERATE
transform            Comp only           ██       CHEAP ✨
opacity              Comp only           ██       CHEAP ✨
```

### The Optimization Principle

**Push work as late in the pipeline as possible.** Composite-only changes are processed by the GPU on a separate thread — they cannot be blocked by JavaScript.

```
❌ Animating left/top: triggers Layout every frame
✅ Animating transform: triggers Composite only — GPU-accelerated

❌ Changing height: triggers full pipeline
✅ Changing transform: skips layout and paint entirely
```

### When to Apply This Model

Apply it whenever:

- You're choosing between CSS properties for animation
- Something triggers unexpected repaints
- You're investigating why a "simple" style change is slow
- You're deciding whether to use `will-change`

`→ Deep dive:` [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md)

---

## 4. The Two-Phase Mental Model

**The model:** Many things in JavaScript happen in two phases: a **setup/creation phase** that runs before execution, and an **execution phase** that runs code line by line.

### Where This Appears

**Execution Contexts:**

```
Phase 1 (Creation):
  - var declarations registered → initialized to undefined
  - function declarations stored in full
  - let/const placed in TDZ

Phase 2 (Execution):
  - Code runs line by line
  - Assignments happen
  - Functions are called
```

**Service Worker:**

```
Phase 1 (Install):
  - Pre-cache critical assets
  - If anything fails → abort, SW goes REDUNDANT

Phase 2 (Activate):
  - Clean up old caches
  - Take control of pages
```

**Browser Rendering:**

```
Phase 1 (Style Recalculation):
  - Compute styles for all elements

Phase 2 (Layout):
  - Compute geometry based on computed styles
```

### The Practical Implication

Two-phase execution is why **hoisting** exists and why **TDZ** exists:

```javascript
// CREATION PHASE:
//   greet → function() {...}  (fully hoisted)
//   name  → undefined         (var hoisted, not initialized)
//   title → <TDZ>             (let hoisted but not initialized)

// EXECUTION PHASE:
console.log(greet("Alice")); // ✅ works — greet was stored in creation phase
console.log(name); // undefined — var initialized to undefined
console.log(title); // ReferenceError — TDZ access

function greet(n) {
  return `Hello, ${n}`;
}
var name = "Alice";
let title = "Engineer";
```

### When to Apply This Model

Apply it when:

- Something is accessible before its declaration (hoisting)
- A Service Worker behaves differently on first load vs subsequent loads
- You're debugging why a function works "before" it's defined
- You're understanding why `let` throws but `var` returns `undefined`

`→ Deep dive:` [`javascript-core/01-execution-context.md`](../javascript-core/01-execution-context.md)

---

## 5. The Lexical vs Dynamic Mental Model

**The model:** In JavaScript, **scope** is lexical (determined by where code is written). But **`this`** binding is dynamic (determined by how a function is called).

### Scope — Always Lexical

```javascript
const name = "outer";

function outer() {
  const name = "inner";
  function read() {
    return name; // reads from WHERE read() is DEFINED — inside outer
  }
  return read;
}

const fn = outer();
fn(); // 'inner' — scope chain set at DEFINITION, not call site
```

The scope chain is established when a function is created, not when it's called. This is the foundation of closures.

### `this` — Dynamic (Except Arrow Functions)

```javascript
const obj = {
  name: "Alice",
  greet() {
    return this.name; // this = determined at CALL TIME
  },
};

obj.greet(); // this = obj → 'Alice'
const fn = obj.greet;
fn(); // this = window/undefined → undefined
```

`this` is determined by the call site:

- `obj.method()` → `this` = `obj`
- `fn()` → `this` = `window` (non-strict) / `undefined` (strict)
- `new Fn()` → `this` = new object
- `fn.call(ctx)` → `this` = `ctx`

### Arrow Functions — Lexical `this`

Arrow functions don't have their own `this`. They inherit it from their lexical context at definition time:

```javascript
const obj = {
  name: "Alice",
  greetLater() {
    setTimeout(() => {
      // Arrow function: this = obj (lexical — captured from greetLater)
      return this.name; // 'Alice' ✅
    }, 100);
  },
};
```

### The Practical Test

When debugging a `this` issue, ask yourself:

1. _Where is this function defined?_ → determines scope (closures, variables)
2. _How is this function called?_ → determines `this`

`→ Deep dive:` [`javascript-core/01-execution-context.md`](../javascript-core/01-execution-context.md)

---

## 6. The Batching Mental Model

**The model:** Systems that process changes individually are slower than systems that collect many changes and process them all at once. Batching is the most universally applicable performance optimization.

### Where Batching Matters

**DOM writes — browser already batches, unless you break it:**

```javascript
// ❌ BREAKS batching: write → read → write → read
// Forces 3 separate layout calculations
div.style.width = "100px"; // invalidates layout
div.offsetWidth; // forces layout NOW
div.style.height = "100px"; // invalidates layout again
div.offsetHeight; // forces layout NOW again

// ✅ PRESERVES batching: all reads, then all writes
const w = div.offsetWidth; // one layout
const h = div.offsetHeight; // (already computed)
div.style.width = w + 10 + "px"; // invalidates once
div.style.height = h + 10 + "px"; // still invalid (no new calculation yet)
// browser recalculates once before next paint
```

**React state — batches multiple setState into one render:**

```javascript
// Both updates render in ONE pass, not two
setCount((c) => c + 1);
setName("Bob");
// → one re-render with both changes applied
```

**Event handlers — all DOM mutations in one handler = one render:**

```javascript
button.addEventListener("click", () => {
  element.style.background = "red"; // not painted yet
  element.style.color = "white"; // not painted yet
  element.textContent = "Clicked"; // not painted yet
  // All three changes applied in ONE paint after handler returns
});
```

### The Universal Batching Pattern

```
1. COLLECT changes (don't process immediately)
2. PROCESS all collected changes at once
3. APPLY the final result
```

This pattern appears in:

- FastDOM (batch DOM reads and writes)
- Redux (batch dispatches)
- `Promise.allSettled` (collect all results)
- Database transactions (collect operations, commit once)
- Object pooling (collect releases, process together)

### When to Apply This Model

Apply it whenever:

- Something processes changes one at a time in a loop
- You're alternating reads and writes to the DOM
- Many small operations cause visible lag
- A system recalculates on every tiny change

`→ Deep dive:` [`performance/03-layout-thrashing.md`](../performance/03-layout-thrashing.md)

---

## 7. The Delegation Mental Model

**The model:** Instead of each object having its own copy of behavior, objects delegate to a shared prototype. Instead of each component handling its own events, a parent handles events on behalf of all children.

### JavaScript Prototypes — Delegation, Not Copying

```
COPY model (classical inheritance):
  Dog class copies all methods from Animal class
  Each Dog instance has its own copy of bark(), eat(), sleep()

DELEGATION model (JavaScript):
  Dog instances DELEGATE to Dog.prototype
  Dog.prototype DELEGATES to Animal.prototype
  ONE copy of each method shared by all instances

1,000,000 Dog instances:
  Copy model: 3 methods × 1,000,000 = 3,000,000 function objects
  Delegation model: 3 methods × 1 = 3 function objects
```

### DOM Event Delegation

```javascript
// ❌ Individual listeners — N listeners for N items
items.forEach((item) => {
  item.addEventListener("click", handler); // one listener per item
});

// ✅ Delegated listener — ONE listener handles all items
list.addEventListener("click", (event) => {
  const item = event.target.closest(".item");
  if (item) handler(item);
});
// Works for dynamically added items too
```

### The Delegation Principle

> **Put behavior where it can be shared. Let lookup find it.**

- Methods on the prototype, data on the instance
- One event listener on the parent, not N listeners on children
- Shared caches at module scope, not per-instance

### When to Apply This Model

Apply it when:

- Creating many objects that share methods
- Adding event listeners to dynamic lists
- Deciding where in an object hierarchy to put a method
- Wondering why `Object.prototype.toString` is available on every object

`→ Deep dive:` [`javascript-core/06-prototypes.md`](../javascript-core/06-prototypes.md)

---

## 8. The Cost Model

**The model:** Every operation has a cost. Some costs are invisible until scale. Knowing the relative costs of common operations lets you make fast architectural decisions.

### The Cost Hierarchy

```
CHEAPEST → MOST EXPENSIVE

Reading a local variable:       ~0.000001ms  (register access)
Reading an object property:     ~0.00001ms   (one pointer follow)
Calling a function:             ~0.0001ms    (stack frame setup)
String concatenation (short):   ~0.001ms
Array.push():                   ~0.001ms
DOM property read (offsetWidth): ~0.1ms      (may force layout)
Style recalculation:            ~0.5ms
Layout (simple):                ~2ms
Layout (complex, many elements): ~20ms
Paint (simple):                 ~2ms
Paint (complex/shadow):         ~10ms
fetch() network request:        ~100-1000ms
```

### The DOM Cost

```
The DOM is a tree managed by the browser's C++ engine.
Every DOM operation crosses the JS→C++ bridge.
That bridge has overhead.

Reading  offsetWidth:  ~0.1ms    (forces potential layout)
Creating a div:        ~0.05ms
Appending to DOM:      ~0.1-2ms  (may trigger layout)
innerHTML (large):     ~10-100ms (parse HTML, rebuild tree)

× 10,000 operations in a loop:
  0.1ms × 10,000 = 1,000ms = frozen UI
```

### The Memory Cost

```
Empty object:        ~56 bytes
Each property:       ~8 bytes
Array (per element): ~8 bytes
Function:            ~100-300 bytes
DOM element:         ~1000+ bytes (JS heap + native memory)

× 1,000,000 objects = 56MB minimum
  (plus property storage, closures, GC overhead)
```

### Using the Cost Model

When reviewing code, ask:

- "How many times does this operation run?"
- "Does it scale with data size, user count, or time?"
- "Is this in a hot path (called every frame, every keystroke)?"

```javascript
// Small cost × many invocations = large real cost
requestAnimationFrame(() => {
  elements.map((el) => el.getBoundingClientRect()); // each: ~0.1ms
  // × 100 elements × 60fps = 600ms of layout per second
});
```

### When to Apply This Model

Apply it before optimization decisions:

- "Is this worth optimizing?" → measure the cost × frequency
- "Should this be a Web Worker?" → is the cost > 5ms?
- "Should I virtualize this list?" → are there > 100 items?

---

## 9. The Tradeoff Mental Model

**The model:** Every architectural decision involves tradeoffs. There are no universally right answers — only choices with different strengths and weaknesses that fit different contexts.

### The Tradeoff Axes

Most frontend engineering tradeoffs live on these axes:

```
Performance  ←──────────────────────→  Developer Experience
Simplicity   ←──────────────────────→  Flexibility
Memory       ←──────────────────────→  Speed
Coupling     ←──────────────────────→  Coordination
Correctness  ←──────────────────────→  Development speed
```

### Example Tradeoffs

**Cache First vs Network First (Service Worker):**

```
Cache First:
  + Fastest possible response (zero network wait)
  + Works offline
  - May serve stale content
  - Must explicitly invalidate

Network First:
  + Always fresh content
  - Slower (network roundtrip)
  - Fails when offline (unless fallback configured)
```

**Micro-Frontends vs Monolith:**

```
Micro-Frontends:
  + Independent deployability
  + Team autonomy
  + Technology flexibility
  - Increased complexity (routing, shared state, auth)
  - Larger total bundle size
  - Harder to maintain consistency

Monolith:
  + Simpler architecture
  + Easier refactoring (one codebase)
  + Smaller bundle (shared dependencies)
  - Deployments are coupled
  - Teams must coordinate
  - Tech stack locked
```

**Object Pool vs Normal Allocation:**

```
Object Pool:
  + Zero GC pressure in hot paths
  + Predictable memory usage
  - Complex code (acquire/release lifecycle)
  - Objects must be properly reset on release
  - Wasted memory if pool is oversized

Normal Allocation:
  + Simple code
  + No manual lifecycle
  - GC pressure in hot paths
  - Potential jank during GC pauses
```

### The Senior Engineer Pattern

When asked about a technical choice, the senior engineer pattern is:

1. **State the tradeoffs, not just the answer**
2. **Ask about the context** ("How many items? How often called? What's the latency budget?")
3. **Pick the simpler solution unless the cost is proven** ("I'd start with X, profile, and switch to Y if needed")

`→ Deep dive:` [`system-design/08-design-tradeoffs.md`](../system-design/08-design-tradeoffs.md)

---

## 10. The Ownership Mental Model

**The model:** Every resource (memory, event listener, timer, subscription, network request) has an owner. The owner is responsible for cleanup when the resource is no longer needed.

### Why Ownership Matters

Without clear ownership, resources leak. The owner is whoever creates the resource — and they must ensure it's cleaned up.

```
Resource created by:          Cleaned up by:
────────────────────────────────────────────────
addEventListener(target, fn)  removeEventListener(target, fn)
setInterval(fn, ms)          clearInterval(id)
new MutationObserver(fn)      observer.disconnect()
store.subscribe(fn)           the returned unsubscribe fn
new Worker(url)               worker.terminate()
fetch() with AbortController  controller.abort()
```

### The Ownership Pattern

Every class that creates resources should implement a `destroy()` method:

```javascript
class Component {
  constructor() {
    // Create → own it
    this._handler = (e) => this.onClick(e);
    document.addEventListener("click", this._handler);

    this._interval = setInterval(() => this.update(), 1000);

    this._observer = new IntersectionObserver((entries) =>
      this.onVisible(entries),
    );
    this._observer.observe(this.el);
  }

  destroy() {
    // Destroy → clean up everything we own
    document.removeEventListener("click", this._handler);
    clearInterval(this._interval);
    this._observer.disconnect();
  }
}
```

### Ownership in Closures

Closures "own" the variables they capture. When a closure outlives its usefulness but still holds a reference:

```javascript
// ❌ Event listener owns a closure that owns `bigData`
// bigData lives as long as the button is in the DOM
function setup(button) {
  const bigData = loadData(); // 10MB
  button.addEventListener("click", () => {
    render(bigData); // closure captures bigData
  });
  // When should bigData be released? Nobody knows.
}

// ✅ Extract minimum needed, release the rest
function setup(button) {
  const bigData = loadData();
  const summary = summarize(bigData); // extract only what's needed
  // bigData goes out of scope → eligible for GC after setup() returns
  button.addEventListener("click", () => {
    render(summary); // only captures summary (small)
  });
}
```

### When to Apply This Model

Apply it when:

- Creating any resource that requires explicit cleanup
- A component or module is "destroyed" but memory doesn't decrease
- Designing APIs — does your API make it easy for callers to clean up?
- Reviewing code — who owns this timer/listener/subscription?

`→ Deep dive:` [`performance/05-memory-leaks.md`](../performance/05-memory-leaks.md)

---

## 11. The Isolation Mental Model

**The model:** Systems are more maintainable when failures, side effects, and state changes are isolated and contained. Isolation limits blast radius.

### Where Isolation Appears

**Scope isolation:**

```javascript
// Variables isolated to their scope — can't affect code outside
{
  const temp = heavyComputation();
  use(temp);
} // temp goes out of scope — GC-eligible, can't accidentally be mutated
```

**Module isolation:**

```javascript
// module.js
let privateState = 0; // isolated to this module — can't be accessed outside
export function increment() {
  privateState++;
}
export function get() {
  return privateState;
}
// External code can't corrupt privateState
```

**Worker isolation:**

```javascript
// Workers have their own heap — a crash in a Worker can't corrupt
// the main thread's memory. Memory pressure in a Worker doesn't
// cause GC pauses on the main thread.
```

**CSS containment:**

```css
.widget {
  contain: layout; /* changes inside cannot affect layout outside */
}
/* A slow mutation inside .widget doesn't trigger layout for the whole page */
```

**Error isolation in observers:**

```javascript
// ❌ One failing observer crashes all others
subscribers.forEach((fn) => fn(data)); // if fn throws, others don't run

// ✅ Isolated error handling
subscribers.forEach((fn) => {
  try {
    fn(data);
  } catch (err) {
    reportError(err);
  } // one failure doesn't block others
});
```

### The Blast Radius Principle

> **Design systems so that when something fails, as little as possible breaks.**

- Use try/catch in observer callbacks so one bad subscriber doesn't break others
- Use `Promise.allSettled` instead of `Promise.all` when partial failure is acceptable
- Use CSS containment so one widget's layout doesn't trigger a full page reflow
- Use Workers so CPU-intensive tasks don't starve the UI thread

### When to Apply This Model

Apply it when:

- Designing module boundaries ("what can this module affect?")
- Writing observer/event systems (isolate subscriber errors)
- Optimizing rendering (CSS containment, compositor layers)
- Deciding between monolith and micro-frontends

---

## 12. The Scale Mental Model

**The model:** Code that works for 10 items, 10 users, or 10 event listeners may fail catastrophically for 10,000 of them. Always ask: "What happens when this grows by 100×?"

### Where Scale Breaks Things

**Event listeners at scale:**

```javascript
// Works fine with 5 items
items.forEach((item) => item.addEventListener("click", handler));

// With 10,000 items:
// - 10,000 event listeners in memory
// - 10,000 references keeping closures alive
// - Browser slower to dispatch events (must check all listeners)
// Scale solution: event delegation — 1 listener handles all
```

**Unbounded caches:**

```javascript
// Works fine for first 100 users
const cache = new Map();
cache.set(userId, data); // grows without bound

// After 1,000,000 unique users: cache = GB of RAM
// Scale solution: LRU cache with size limit
```

**Nested loops:**

```javascript
// O(n²) — fine for n=100, catastrophic for n=10,000
items.forEach((a) => {
  items.forEach((b) => {
    compare(a, b); // 100 × 100 = 10,000 ops
    // 10,000 × 10,000 = 100,000,000 ops
  });
});
```

**Synchronous processing:**

```javascript
// Works for 100 items
items.forEach((item) => process(item)); // 100 × 1ms = 100ms (borderline)

// For 10,000 items: 10,000 × 1ms = 10,000ms = frozen UI
// Scale solution: virtualization + chunked processing
```

### The Scale Questions

Before finalizing any design, ask:

```
1. What is the current scale? (items, users, events, calls/second)
2. What is the expected maximum scale?
3. What is the complexity (O(n)? O(n²)? O(1))?
4. At what scale does this break? (10×? 100×? 1000×?)
5. What's the fix if it breaks? (Do we need it now or can we defer?)
```

### The Rule of Premature Optimization

> **Measure first, optimize second.**

The scale mental model is not about optimizing everything upfront. It's about:

1. **Recognizing** when code has a scaling problem
2. **Estimating** at what scale it becomes a real problem
3. **Deciding** whether that scale is plausible for your use case
4. **Planning** the refactor path if it becomes necessary

Sometimes `O(n²)` is fine because `n` never exceeds 50 in practice. Sometimes `O(n)` is catastrophic because `n` is 10 million. Know your scale.

`→ Deep dive:` [`system-design/01-large-scale-architecture.md`](../system-design/01-large-scale-architecture.md)

---

## Applying the Models Together

Real engineering problems require multiple models simultaneously. Here's an example:

**Problem:** "Our dashboard is slow when it has 200 widgets."

Apply the models:

```
Pipeline model:
  → Each widget update triggers layout + paint?
  → Can we reduce to composite-only updates?

Cost model:
  → 200 widgets × layout cost = significant
  → Are any triggering layout thrashing?

Batching model:
  → Are widget updates batched or processed individually?
  → Can we defer all updates to one RAF flush?

Scale model:
  → Does the pattern work for 20 widgets?
  → At 200 it's slow — what's the O() complexity?

Isolation model:
  → Can CSS containment isolate each widget's layout?
  → Does one slow widget block the others?

Single-thread model:
  → Can widget data processing move to a Worker?
  → What's left on the main thread?
```

No single model gives you the answer. Together, they give you the diagnosis.

---

## The Meta-Model: Think in Invariants

A master-level mental model: **think in invariants** — things that are always true regardless of the specific situation.

```
Invariant: The GC can only collect what is unreachable.
  → Therefore: any memory leak has a reference you didn't remove.
  → You don't need to understand the GC algorithm to find leaks.
  → Just find the unexpected reference.

Invariant: The main thread can only do one thing at a time.
  → Therefore: any frozen UI has a long-running synchronous operation.
  → You don't need a profiler to know this — the model tells you.
  → Just find the synchronous bottleneck.

Invariant: The pipeline is sequential.
  → Therefore: if composite is slow, layout must have triggered.
  → If layout is slow, a geometry property changed.
  → Work backwards from the symptom to the cause.
```

Invariants let you reason about systems you haven't profiled yet. They let you make predictions without running the code.

---

_These mental models are the foundation. Every deep dive in this handbook builds on them. When something doesn't make sense in a later section, come back here — the answer is usually in one of these twelve models._
