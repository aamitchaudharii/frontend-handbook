# 08 — Memory Management

> **"JavaScript manages memory for you — but only for objects it knows you're done with. The ones you forget to release? Those are yours forever."**

Memory management in JavaScript is automatic, but not magical. The garbage collector can only collect what it can prove is unreachable. Everything else stays alive — consuming RAM, increasing GC pause times, and eventually degrading your application. This document covers how JS memory works from the ground up: allocation, the heap, garbage collection strategies, and every pattern that prevents memory from being released.

---

## 📚 Table of Contents

1. [How JavaScript Allocates Memory](#1-how-javascript-allocates-memory)
2. [The Stack vs The Heap](#2-the-stack-vs-the-heap)
3. [Memory Lifecycle](#3-memory-lifecycle)
4. [Garbage Collection — The Reachability Model](#4-garbage-collection--the-reachability-model)
5. [GC Roots](#5-gc-roots)
6. [Reference Counting — Why JS Doesn't Use It Alone](#6-reference-counting--why-js-doesnt-use-it-alone)
7. [Mark-and-Sweep — The Core Algorithm](#7-mark-and-sweep--the-core-algorithm)
8. [Generational GC in V8](#8-generational-gc-in-v8)
9. [GC Pauses and Their Impact](#9-gc-pauses-and-their-impact)
10. [What Prevents GC — Common Retention Patterns](#10-what-prevents-gc--common-retention-patterns)
11. [WeakRef and FinalizationRegistry](#11-weakref-and-finalizationregistry)
12. [WeakMap and WeakSet for Memory-Safe Associations](#12-weakmap-and-weakset-for-memory-safe-associations)
13. [Measuring Memory](#13-measuring-memory)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. How JavaScript Allocates Memory

Every time you create a value in JavaScript, memory is allocated. This happens automatically — you never call `malloc`. But knowing _what_ triggers allocation and _where_ it goes is essential for understanding performance and leaks.

### What Triggers Allocation

```javascript
// Primitive values — allocated on stack (or inlined)
const n = 42; // number: 8 bytes (64-bit float)
const b = true; // boolean: 1 byte
const s = "hello"; // string: heap (reference on stack)

// Objects — allocated on heap
const obj = {}; // empty object: ~64 bytes minimum
const arr = [1, 2, 3]; // array: ~100 bytes + element storage
const fn = () => {}; // function: ~100 bytes + closure env
const re = /pattern/gi; // regex: ~80 bytes
const map = new Map(); // map: ~200 bytes

// DOM nodes — heap (JS heap + native memory)
const el = document.createElement("div"); // ~1000+ bytes

// TypedArrays — contiguous heap memory
const buf = new ArrayBuffer(1024 * 1024); // exactly 1MB
const f32 = new Float32Array(1000); // 4000 bytes (4 bytes × 1000)
```

### Object Size Estimation

```javascript
// Rough size estimates in V8:
// Empty object:           ~56 bytes
// Each property:          ~8 bytes (pointer)
// String (per char):      ~2 bytes (UTF-16) + header (~32 bytes)
// Number (heap-allocated): ~32 bytes
// Array (per element):    ~8 bytes (pointer) + header
// Function:               ~100–300 bytes + captured scope size
// Closure environment:    size of all captured variables
```

---

## 2. The Stack vs The Heap

Memory in a JavaScript program is divided between the **call stack** (fast, limited, automatic) and the **heap** (large, GC-managed, flexible).

### The Call Stack

```
Stack characteristics:
  - Fixed size per thread (~1–8MB in V8)
  - LIFO: automatically reclaimed when function returns
  - Stores: local primitive variables, references (pointers to heap objects)
  - Fast allocation/deallocation (just move a pointer)
  - No garbage collection needed — managed by call stack discipline

What lives on the stack:
  - Primitive variable values (number, boolean, null, undefined)
  - References (pointers) to heap objects
  - Function execution metadata (return address, frame pointer)
```

### The Heap

```
Heap characteristics:
  - Dynamic size (up to system RAM / browser limits)
  - Unordered allocation — objects can be anywhere
  - Managed by the Garbage Collector
  - Slower allocation than stack (must find free space)
  - Objects persist across function calls (as long as referenced)

What lives on the heap:
  - All objects ({}, [], Map, Set, etc.)
  - All functions (including closures)
  - All strings
  - DOM nodes (also have native memory footprint)
  - ArrayBuffers and TypedArrays
```

### Stack Holds References to Heap

```javascript
function example() {
  const num = 42; // ← 42 is ON the stack (primitive)
  const str = "hello"; // ← reference on stack, string on heap
  const obj = { x: 1 }; // ← reference on stack, object on heap

  // When example() returns:
  // - num (42): gone with the stack frame
  // - str reference: gone with stack frame; string on heap eligible for GC
  // - obj reference: gone with stack frame; object on heap eligible for GC
  //   (IF no other references to the object exist)
}
```

```
Stack:                        Heap:
┌──────────────┐             ┌──────────────────────────┐
│ example frame│             │  String: "hello"         │
│  num: 42     │             │  Object: { x: 1 }        │
│  str: ───────┼────────────►│                          │
│  obj: ───────┼────────────►│                          │
└──────────────┘             └──────────────────────────┘
   (frame popped)               (GC decides what to keep)
```

---

## 3. Memory Lifecycle

Every piece of memory goes through three stages:

```
┌─────────────────────────────────────────────────────────┐
│                  MEMORY LIFECYCLE                        │
│                                                          │
│   1. ALLOCATE                                            │
│      Engine reserves memory for the new value           │
│      const obj = {} → ~56 bytes allocated in heap       │
│                                                          │
│   2. USE                                                 │
│      Code reads and writes the memory                   │
│      obj.name = 'Alice' → writes to allocated memory    │
│                                                          │
│   3. RELEASE                                             │
│      Memory is returned to be reused                    │
│      When obj is unreachable → GC reclaims the memory   │
│                                                          │
│   LEAK = step 3 never happens for memory you're done with│
└─────────────────────────────────────────────────────────┘
```

### When Release Happens

Release is triggered by the **Garbage Collector** running a collection cycle. The GC:

1. Identifies all unreachable objects (objects no longer accessible from roots)
2. Reclaims their memory

The developer's job is not to call `free()` — it's to **not hold references** to things they're done with. The GC does the rest.

---

## 4. Garbage Collection — The Reachability Model

V8 uses a **reachability-based garbage collector**. The rule is simple:

> **An object is garbage-collected if and only if it is unreachable from any root.**

If even ONE reference to an object exists anywhere in the root set or transitively reachable from it, the object stays alive — regardless of whether it will ever actually be used again.

```javascript
// Reachable → NOT collected
let obj = { name: "Alice", data: new Array(1_000_000) };
// obj is reachable via the variable `obj` in global scope
// 8MB+ stays alive

// Unreachable → eligible for collection
obj = null;
// obj no longer points to { name: 'Alice' }
// No other references exist → eligible for GC
// GC will reclaim the ~8MB on next collection cycle
```

### The Key Insight

The GC doesn't know your _intent_. It only knows the reference graph. If a reference exists to an object you consider "dead," the object stays alive. Memory leaks are almost always this: **a live reference to a conceptually dead object**.

---

## 5. GC Roots

Roots are the starting points for the GC's reachability traversal. Anything reachable from a root stays alive.

```
GC ROOTS in a browser JavaScript context:

1. Global variables (window, globalThis, and any properties on them)
2. Currently executing function's local variables (call stack frames)
3. JavaScript closure environments (if referenced by any alive function)
4. DOM nodes currently in the document (and any JS references to them)
5. Event listener callbacks registered with the browser
6. setTimeout / setInterval callbacks not yet fired
7. Pending Promise continuations
8. Web Worker message ports
9. References held by browser's internal C++ layer
```

```
Root traversal:

  window.app ──────────────────────────────────► App object
                                                    ├── UserService ──► User[]
                                                    ├── EventBus ────► handlers[]
                                                    └── Router ──────► Route[]

  Call Stack (executing fn) ───────────────────► local variable objects

  Event listeners ─────────────────────────────► callback functions
                                                    └── closure envs
                                                         └── captured objects

Everything reachable via these paths = ALIVE (cannot be GC'd)
Everything NOT reachable via any path = DEAD (eligible for GC)
```

---

## 6. Reference Counting — Why JS Doesn't Use It Alone

**Reference counting** is a simpler GC approach: each object keeps a count of how many references point to it. When the count hits zero, the object is freed immediately.

```
Reference count example:
  const a = {};  // a → {} : refcount = 1
  const b = a;   // b → {} : refcount = 2
  a = null;      // a no longer → {} : refcount = 1
  b = null;      // b no longer → {} : refcount = 0 → freed immediately
```

### The Circular Reference Problem

Reference counting alone **cannot handle cycles**:

```javascript
// Circular reference — both objects have refcount > 0 forever
function createCycle() {
  const objA = {};
  const objB = {};

  objA.ref = objB; // objA → objB: B's refcount = 1
  objB.ref = objA; // objB → objA: A's refcount = 1

  // createCycle returns — local variables dropped
  // But A and B each have refcount = 1 (pointing to each other)
  // Neither will EVER reach 0 → NEVER freed by reference counting
}
createCycle();
// A and B are now unreachable from any root
// But refcount GC can't collect them — they keep each other alive
```

This is why V8 uses **mark-and-sweep** (reachability-based) as its primary algorithm — it handles cycles correctly.

> **Historical note:** IE6/7 used reference counting for COM objects (DOM). This caused notorious memory leaks with circular references between JS objects and DOM elements. Modern browsers solved this with full mark-and-sweep.

---

## 7. Mark-and-Sweep — The Core Algorithm

The mark-and-sweep algorithm operates in two phases:

### Phase 1 — Mark

Starting from all roots, traverse the entire reachability graph and **mark** every reachable object:

```
Mark phase:

  Start: all objects unmarked (white)

  1. Push all roots onto a work queue
  2. For each object in the queue:
     a. Mark it as "reachable" (grey → black)
     b. Add all objects it references to the queue
  3. Repeat until queue is empty

  After mark phase:
    Black objects: reachable → keep
    White objects: unreachable → reclaim
```

### Phase 2 — Sweep

Scan the heap and reclaim all unmarked objects:

```
Sweep phase:

  Walk the entire heap:
    For each object:
      If marked (black): clear mark, keep object
      If unmarked (white): reclaim memory → add to free list

  After sweep:
    All unreachable objects freed
    Free list available for new allocations
```

### Why Mark-and-Sweep Handles Cycles

```javascript
// Both A and B are unreachable from roots
// Mark phase: neither A nor B is reached during traversal
// Sweep phase: both A and B are collected
// ✓ Cycles are handled correctly
```

### Cost of Mark-and-Sweep

- Mark phase: proportional to the number of **live** objects
- Sweep phase: proportional to the **entire heap** size
- Both phases can cause **GC pauses** — the main thread stops while GC runs

Modern V8 mitigates this with incremental marking, concurrent sweeping, and parallel GC — but the fundamental algorithm is mark-and-sweep.

---

## 8. Generational GC in V8

V8 uses a **generational hypothesis**: most objects die young. A newly allocated object is much more likely to become garbage soon than an object that has survived several GC cycles.

This insight drives V8's two-generation design:

```
V8 HEAP REGIONS:

┌──────────────────────────────────────────────────────────────┐
│                    YOUNG GENERATION                           │
│                                                               │
│  ┌─────────────────────┐  ┌─────────────────────┐           │
│  │   Nursery (From)    │  │   Survivor (To)      │           │
│  │   ~1–8MB            │  │   ~1–8MB             │           │
│  │   New allocations   │  │   Survivors go here  │           │
│  └─────────────────────┘  └─────────────────────┘           │
│                                                               │
│  GC: Scavenge (copying collector)                            │
│  Frequency: Very frequent (~every few MB allocated)          │
│  Duration: ~1ms (very fast)                                  │
└──────────────────────────────────────────────────────────────┘
           │ Objects surviving 2+ scavenges are PROMOTED
           ▼
┌──────────────────────────────────────────────────────────────┐
│                    OLD GENERATION                             │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Old Space   (general objects)       up to ~1.5GB     │   │
│  │  Code Space  (compiled JS code)                       │   │
│  │  Map Space   (hidden classes/shapes)                  │   │
│  │  Large Object Space (>256KB objects, not moved)       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  GC: Major GC (Mark-Sweep-Compact)                           │
│  Frequency: Infrequent (when old gen fills up)               │
│  Duration: Varies — 10ms to 100ms+ for large heaps          │
└──────────────────────────────────────────────────────────────┘
```

### Young Generation — Scavenge

```
Scavenge (copying collector):

From-space (live + dead objects):
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ LIVE: A │ DEAD: B │ LIVE: C │ DEAD: D │ LIVE: E │
└─────────┴─────────┴─────────┴─────────┴─────────┘

After scavenge — live objects COPIED to To-space:
┌─────────┬─────────┬─────────┐
│ LIVE: A │ LIVE: C │ LIVE: E │  ← compact, no fragmentation
└─────────┴─────────┴─────────┘

From-space entirely reclaimed.
Swap: old To-space becomes new From-space for next cycle.
```

**Why copying?** No fragmentation — live objects are always packed together. New allocations just bump a pointer. Very fast.

### Old Generation — Major GC

Objects that survive 2 Scavenge cycles are promoted to old generation. The old generation uses Mark-Sweep-Compact:

1. **Mark**: Traverse from roots, mark all reachable objects
2. **Sweep**: Reclaim unreachable objects
3. **Compact** (optional): Move live objects together to reduce fragmentation

Major GC is expensive because:

- Traverses the entire old generation
- The old generation can be hundreds of MB or GB
- Compaction requires moving objects and updating all references

### Incremental and Concurrent GC

V8 mitigates GC pauses with:

- **Incremental marking**: Splits mark phase into small increments, interleaved with JS execution
- **Concurrent marking**: Marks from additional threads while JS runs
- **Concurrent sweeping**: Sweeps on background threads
- **Parallel GC**: Uses multiple threads for both mark and sweep phases

Despite these improvements, large heaps with many live objects still cause noticeable pauses. The best optimization: don't accumulate a large heap.

---

## 9. GC Pauses and Their Impact

When the GC runs, it may need to stop JavaScript execution temporarily. This is called a **stop-the-world pause**.

```
Normal frame (no GC):
[JS: 4ms] [Layout: 2ms] [Paint: 2ms] = 8ms ✓ (within 16ms budget)

Frame with minor GC (Scavenge):
[JS: 4ms] [GC Scavenge: 1ms] [Layout: 2ms] [Paint: 2ms] = 9ms ✓ (still OK)

Frame with major GC:
[JS: 4ms] [GC Major: 60ms!!] [Layout: 2ms] [Paint: 2ms] = 68ms ✗
                                                            (4 frames dropped!)
```

### What Triggers Major GC

- Old generation fills up (most common)
- Explicit `window.gc()` call (dev flag only)
- Browser memory pressure from other tabs

### Monitoring GC

```javascript
// Observe GC via PerformanceObserver
// (Note: specific GC entries require --enable-precise-memory-info flag in Chrome)

// Indirect measurement: memory changes
function monitorMemory(intervalMs = 5000) {
  if (!performance.memory) return;

  let lastHeap = performance.memory.usedJSHeapSize;

  setInterval(() => {
    const current = performance.memory.usedJSHeapSize;
    const diff = current - lastHeap;
    const diffMB = (diff / 1024 / 1024).toFixed(2);

    console.log(
      `Heap: ${(current / 1024 / 1024).toFixed(1)}MB`,
      diff > 0 ? `(+${diffMB}MB)` : `(${diffMB}MB — GC ran)`,
    );

    lastHeap = current;
  }, intervalMs);
}

monitorMemory();
```

### Detecting Long GC Pauses with rAF

```javascript
// Large gaps between rAF callbacks indicate GC pauses or long tasks
let lastFrame = performance.now();

requestAnimationFrame(function checkPauses(now) {
  const gap = now - lastFrame;

  if (gap > 50) {
    console.warn(
      `Long pause detected: ${gap.toFixed(1)}ms`,
      "— possible GC pause or long task",
    );
  }

  lastFrame = now;
  requestAnimationFrame(checkPauses);
});
```

---

## 10. What Prevents GC — Common Retention Patterns

These patterns create references that keep objects alive after they're conceptually dead. Each one is a potential memory leak.

### 1. Global Variables

```javascript
// ❌ Global variables live for the entire page lifetime
window.cache = new Map(); // grows forever
window.userSessions = []; // grows forever

// ❌ Accidental global (no declaration keyword)
function init() {
  appState = { user: null, data: [] }; // becomes window.appState
}
```

### 2. Closures Capturing Large Objects

```javascript
// ❌ 8MB array kept alive by tiny callback
function setup(element) {
  const hugeData = loadDataset(); // 8MB

  element.addEventListener("click", () => {
    // closure captures entire scope of setup()
    // hugeData is in that scope → stays alive
    doSomething();
  });
}
```

### 3. Detached DOM Nodes

```javascript
// ❌ DOM removed but JS reference still held
const detachedNodes = [];

function removeEl(el) {
  el.remove(); // removed from document
  detachedNodes.push(el); // reference kept → entire subtree stays in memory
}
```

### 4. Forgotten Timers

```javascript
// ❌ setInterval keeps callback alive forever
class LiveUpdate {
  constructor() {
    this.data = new Array(100_000);
    setInterval(() => {
      this.refresh(); // `this` captured → LiveUpdate + data stay alive
    }, 1000);
    // Even if LiveUpdate "goes away", the interval keeps it alive
  }
}
```

### 5. Retained Event Listeners

```javascript
// ❌ Listener on long-lived element captures short-lived component
class Modal {
  constructor() {
    this.data = loadModalData(); // large dataset
    // document is a long-lived element
    document.addEventListener("keydown", this.handleKey.bind(this));
    // document holds reference to bound function → bound fn holds `this` → Modal stays alive
  }
  // If Modal is never explicitly destroyed, it and this.data live forever
}
```

### 6. Console.log References

```javascript
// ❌ DevTools keeps references to console.log'd objects alive
//    (so you can inspect them later in the console panel)
console.log(largeObject); // DevTools holds a reference
// This is a development concern, not production, but confuses heap profiling
```

### 7. Out-of-DOM Event Delegation

```javascript
// ❌ Event delegation on removed containers
const handlers = new Map(); // grows unboundedly

function addHandler(id, fn) {
  handlers.set(id, fn); // fn captures component data
}

// Component destroyed but handler still in map
function removeComponent(id) {
  destroyComponent(id);
  // forgot: handlers.delete(id)
  // fn and captured data stay alive in handlers
}
```

---

## 11. WeakRef and FinalizationRegistry

`WeakRef` allows holding a reference to an object **without preventing its garbage collection**.

### WeakRef

```javascript
// WeakRef: hold reference without preventing GC
let user = { name: "Alice", data: new Array(100_000) };

const weakRef = new WeakRef(user);

// Normal usage: dereference
const currentUser = weakRef.deref();
if (currentUser) {
  console.log(currentUser.name); // 'Alice'
} else {
  console.log("User was garbage collected");
}

// Drop strong reference
user = null;

// Now the object is eligible for GC
// weakRef.deref() may return undefined at any point after this
```

### FinalizationRegistry

Gets notified when a weakly-held object is garbage collected:

```javascript
const registry = new FinalizationRegistry((heldValue) => {
  // Called after the registered object is GC'd
  // heldValue is whatever you passed as the second arg to register()
  console.log(`Object '${heldValue}' was garbage collected`);
  cleanupResources(heldValue);
});

function createComponent(name) {
  const component = { name, data: new Array(10_000) };

  // Register component for cleanup notification
  // 'name' is the held value — passed to the callback
  // It must NOT reference `component` (would prevent GC)
  registry.register(component, name);

  return component;
}

let comp = createComponent("dashboard");
comp = null; // drop reference — component eligible for GC
// Later: "Object 'dashboard' was garbage collected" logged
```

### When to Use WeakRef

```javascript
// ✅ Cache that doesn't prevent GC of its entries
class WeakCache {
  constructor() {
    this._cache = new Map(); // key → WeakRef<value>
    this._registry = new FinalizationRegistry((key) => {
      this._cache.delete(key); // clean up when value is GC'd
    });
  }

  set(key, value) {
    this._cache.set(key, new WeakRef(value));
    this._registry.register(value, key, value);
  }

  get(key) {
    const ref = this._cache.get(key);
    if (!ref) return undefined;
    const value = ref.deref();
    if (!value) {
      this._cache.delete(key); // already GC'd
      return undefined;
    }
    return value;
  }
}
```

**Important caveats:**

- `deref()` can return `undefined` at ANY point — always check
- Don't use WeakRef as a performance optimization for avoiding memory
- The exact timing of GC is implementation-dependent — don't rely on it
- FinalizationRegistry callbacks run after GC, possibly on a separate microtask

---

## 12. WeakMap and WeakSet for Memory-Safe Associations

`WeakMap` and `WeakSet` hold their keys **weakly** — if the key object is garbage collected, the entry is automatically removed.

### WeakMap — Attaching Data to Objects Without Preventing GC

```javascript
// ✅ Associate metadata with DOM elements
// When the element is removed and GC'd, the metadata is automatically removed
const elementMeta = new WeakMap();

function setMeta(element, data) {
  elementMeta.set(element, data);
}

function getMeta(element) {
  return elementMeta.get(element);
}

// Usage:
const btn = document.createElement("button");
setMeta(btn, { clickCount: 0, created: Date.now() });

// Later:
btn.remove(); // removed from DOM
// If no other JS references to btn exist, it's GC'd
// elementMeta entry is automatically removed — no manual cleanup needed
```

### WeakMap for Private Class Data (Before #private fields)

```javascript
// WeakMap pattern for private data (now superseded by #private fields)
const _private = new WeakMap();

class SecureStore {
  constructor(initialData) {
    _private.set(this, {
      data: initialData,
      accessLog: [],
    });
  }

  get(key) {
    const priv = _private.get(this);
    priv.accessLog.push({ key, time: Date.now() });
    return priv.data[key];
  }

  getAccessLog() {
    return [..._private.get(this).accessLog];
  }
}

const store = new SecureStore({ secret: "xyz" });
store.get("secret"); // 'xyz'
store._private; // undefined — not accessible
```

### WeakSet — Tracking Objects Without Holding Them

```javascript
// ✅ Track which elements have been processed without preventing GC
const processedElements = new WeakSet();

function processOnce(element) {
  if (processedElements.has(element)) {
    return; // already processed
  }
  processedElements.add(element);
  doProcessing(element);
  // When element is removed from DOM and GC'd,
  // the WeakSet entry is automatically removed — no cleanup needed
}
```

### WeakMap vs Map for Associations

| Use case                 | Map                | WeakMap                    |
| ------------------------ | ------------------ | -------------------------- |
| Key is a primitive       | ✅ Only Map can    | ❌ Keys must be objects    |
| Need to iterate entries  | ✅ Map is iterable | ❌ WeakMap is not iterable |
| Need `.size`             | ✅                 | ❌                         |
| Key object may be GC'd   | ❌ Map prevents GC | ✅ WeakMap allows GC       |
| Metadata on DOM elements | ❌ Use WeakMap     | ✅                         |
| Cache keyed by objects   | ❌ Use WeakMap     | ✅                         |

---

## 13. Measuring Memory

### Chrome DevTools — Memory Tab

```
THREE KEY TOOLS:

1. Heap Snapshot
   → Point-in-time view of all live objects
   → Shows object count, shallow size, retained size
   → Use: compare before/after action to find what grew

2. Allocation Instrumentation on Timeline
   → Records allocations over time
   → Shows which allocations were NOT collected (blue = live)
   → Use: find ongoing leaks in animations/event handlers

3. Allocation Sampling
   → Lightweight profiler of allocation call sites
   → Shows which JS functions allocate the most
   → Use: find hot allocation paths for optimization
```

### Heap Snapshot Terms

```
Shallow Size:
  Memory directly occupied by the object itself
  (not including objects it references)
  Example: { name: 'Alice' } — ~56 bytes (pointers to string)

Retained Size:
  Total memory freed if this object were GC'd
  = object's shallow size + all objects it uniquely holds alive
  Example: { name: 'Alice', data: [1M items] } — retained = ~8MB

Distance:
  Steps from GC root to this object
  Lower distance = closer to root = more likely to be important

Retainers:
  The objects/references keeping this object alive
  = the path from a root to this object
```

### Programmatic Memory Measurement

```javascript
// Basic heap stats (Chrome only)
function heapStats() {
  if (!performance.memory) {
    console.warn("performance.memory not available");
    return null;
  }
  return {
    used: Math.round(performance.memory.usedJSHeapSize / 1024 / 1024) + "MB",
    total: Math.round(performance.memory.totalJSHeapSize / 1024 / 1024) + "MB",
    limit: Math.round(performance.memory.jsHeapSizeLimit / 1024 / 1024) + "MB",
  };
}

// Before/after comparison
async function measureMemoryImpact(action, label = "action") {
  // Force GC if available (requires --js-flags="--expose-gc" in Node,
  // or chrome://flags > "Enable precise memory info" in Chrome)
  if (typeof gc === "function") gc();
  await new Promise((r) => setTimeout(r, 100)); // let GC settle

  const before = performance.memory?.usedJSHeapSize ?? 0;

  await action();

  if (typeof gc === "function") gc();
  await new Promise((r) => setTimeout(r, 100));

  const after = performance.memory?.usedJSHeapSize ?? 0;
  const delta = ((after - before) / 1024 / 1024).toFixed(2);

  console.log(`[Memory] ${label}: ${delta > 0 ? "+" : ""}${delta}MB`);
  return { before, after, delta: after - before };
}

// Usage
await measureMemoryImpact(async () => {
  const component = new HeavyComponent();
  await component.load();
  component.destroy(); // verify destroy releases memory
}, "HeavyComponent lifecycle");
```

---

## 14. Good Practices

### ✅ Null out references when done with large objects

```javascript
class DataProcessor {
  async process(rawData) {
    const parsed = await parse(rawData); // possibly large
    const result = transform(parsed);

    // parsed is no longer needed — null it out
    // GC can reclaim it even if `this` stays alive
    // (V8 may do this automatically via dead variable analysis,
    //  but explicit null is clear and reliable)
    parsed = null;

    return result;
  }
}
```

### ✅ Use WeakMap/WeakSet for object-keyed associations

```javascript
// ✅ Metadata attached to elements — auto-cleaned when element GC'd
const meta = new WeakMap();

element.addEventListener("click", handler);
meta.set(element, { handler, registeredAt: Date.now() });

// When element is removed and GC'd, meta entry disappears automatically
```

### ✅ Use `const` to signal immutable bindings — aids dead code analysis

```javascript
// const signals: this binding will never be reassigned
// V8 can use this for optimization
const MAX_RETRIES = 3;
const BASE_URL = "https://api.example.com";
```

### ✅ Minimize what closures capture

```javascript
// ❌ Closure captures large scope
function setup(bigData) {
  element.addEventListener("click", () => {
    process(bigData); // captures entire bigData (possibly MB)
  });
}

// ✅ Extract only what's needed
function setup(bigData) {
  const processedOnce = preprocess(bigData); // extract what matters
  bigData = null; // explicitly release (optional — depends on context)

  element.addEventListener("click", () => {
    use(processedOnce); // only processedOnce captured — smaller
  });
}
```

### ✅ Always pair registrations with cleanup

```javascript
class Component {
  mount() {
    this._resizeObserver = new ResizeObserver(this._onResize.bind(this));
    this._resizeObserver.observe(this.el);

    this._unsubscribe = store.subscribe(this._onStoreUpdate.bind(this));

    this._intervalId = setInterval(this._refresh.bind(this), 5000);
  }

  unmount() {
    this._resizeObserver?.disconnect();
    this._resizeObserver = null;

    this._unsubscribe?.();
    this._unsubscribe = null;

    clearInterval(this._intervalId);
    this._intervalId = null;
  }
}
```

---

## 15. Bad Practices

### ❌ Growing collections without eviction

```javascript
// ❌ Grows forever — every API response cached forever
const responseCache = new Map();
async function fetchData(key) {
  if (!responseCache.has(key)) {
    responseCache.set(key, await fetch(key).then((r) => r.json()));
  }
  return responseCache.get(key);
}

// ✅ Bounded LRU cache
const responseCache = new LRUCache({ maxSize: 200, ttl: 5 * 60 * 1000 });
```

### ❌ Keeping references in module scope to component data

```javascript
// ❌ Module-level array holds component references forever
const allComponents = [];

function createComponent(config) {
  const comp = new Component(config);
  allComponents.push(comp); // never removed
  return comp;
}
```

### ❌ String concatenation in loops (creates many intermediate strings)

```javascript
// ❌ Each + creates a new string allocation — N intermediate strings
let html = "";
for (const item of items) {
  html += `<li>${item.name}</li>`; // N allocations, N-1 immediately garbage
}

// ✅ Array join — one allocation
const html = items.map((item) => `<li>${item.name}</li>`).join("");
```

### ❌ Large objects as default parameter values

```javascript
// ❌ The default object is created ONCE and shared (not per call)
// It's a fixed object, fine memory-wise, but MUTATING it is a bug
function configure(options = { retries: 3, timeout: 5000 }) {
  options.retries--; // mutates the shared default!
}

// ✅ Create a new object per call
function configure(options = {}) {
  const config = { retries: 3, timeout: 5000, ...options };
  config.retries--;
  return config;
}
```

---

## 16. Common Mistakes

### Mistake 1 — Thinking `= null` immediately frees memory

```javascript
let bigObj = loadLargeData();
bigObj = null; // marks bigObj as eligible for GC
// does NOT immediately free memory
// Memory freed when GC next runs — timing not guaranteed

// This is still correct practice — it makes the object eligible for GC
// The actual reclamation happens asynchronously
```

### Mistake 2 — Expecting GC to handle detached DOM nodes

```javascript
const container = document.getElementById("container");
container.innerHTML = ""; // removes children from DOM

// ❌ But if you stored references before clearing:
const children = Array.from(container.children); // captured before clearing
container.innerHTML = "";
// children array still holds references to the detached nodes
// They will NOT be GC'd until children is cleared too
```

### Mistake 3 — Forgetting that closures in event handlers prevent GC

```javascript
function initPage() {
  const pageData = {
    /* large object */
  };

  // pageData stays alive for the lifetime of the page
  // because the click handler closes over it
  document.getElementById("btn").addEventListener("click", () => {
    render(pageData);
  });
}
// Even after initPage() returns, pageData lives forever via the listener
```

### Mistake 4 — Using arrays instead of typed arrays for numeric data

```javascript
// ❌ Regular array of numbers — each number is a heap-allocated Float64
const coords = [];
for (let i = 0; i < 100_000; i++) {
  coords.push(Math.random()); // 100k heap allocations
}
// ~800KB + heap overhead

// ✅ Float64Array — contiguous memory, no per-element allocation
const coords = new Float64Array(100_000);
for (let i = 0; i < coords.length; i++) {
  coords[i] = Math.random(); // write directly to buffer
}
// Exactly 800KB, no heap fragmentation, cache-friendly
```

---

## 17. Interview-Level Explanation

> **"How does memory management work in JavaScript? What causes memory leaks?"**

**Strong answer:**

> "JavaScript uses automatic memory management with a garbage collector. Memory is allocated when you create values — objects, strings, and functions go on the heap; primitive values and references live on the stack. You never call malloc or free — the GC handles reclamation.
>
> V8 uses a generational garbage collector. Most objects are short-lived — they're allocated in the young generation, and a fast copying collector called Scavenge runs frequently and cheaply to reclaim them. Objects that survive multiple Scavenges get promoted to the old generation, where a full Mark-Sweep-Compact algorithm runs less frequently but at higher cost.
>
> The GC uses reachability as its criterion. Starting from roots — global variables, call stack locals, event listeners, timers — it traverses the entire reference graph and marks everything reachable. Anything NOT marked is garbage and gets reclaimed. The key insight: the GC can't collect an object if any reference to it exists from a root, even if that object will never actually be used again. That's what a memory leak is.
>
> The most common leak patterns are: forgotten event listeners where a component adds a window or document listener but never removes it on cleanup; uncleared timers where setInterval callbacks capture component data indefinitely; detached DOM nodes where elements are removed from the document but JS still holds references; unbounded caches that grow without eviction; and closures that capture large objects in long-lived callbacks.
>
> WeakMap and WeakSet are the right tool for associating data with objects when you don't want to prevent GC — they hold keys weakly, so when the key object is collected, the entry disappears automatically. For explicit optional references, WeakRef provides deref() which returns undefined once the object is GC'd."

---

## 18. Exercises

### Exercise 1 — Identify what prevents GC

For each code snippet, identify what prevents the annotated object from being garbage collected:

```javascript
// a) What prevents `userData` from being collected?
function setupDashboard() {
  const userData = fetchUserData(); // large object
  document.getElementById("refresh").addEventListener("click", function () {
    renderUser(userData);
  });
}
setupDashboard();

// b) What prevents `cache` entries from being collected?
const cache = {};
function remember(key, fn) {
  if (!cache[key]) cache[key] = fn();
  return cache[key];
}
remember("config", loadConfig);

// c) What prevents `component` from being collected?
let component = new HeavyComponent();
window._debug = { component };
component = null;
```

<details>
<summary>Answers</summary>

```
a) userData is prevented from being GC'd by:
   → The click event listener callback, which closes over the scope
     of setupDashboard(), which contains userData.
   → document.getElementById('refresh') (a live DOM element) holds
     a reference to the callback → callback's closure → userData.
   Fix: store the handler reference and removeEventListener on cleanup,
   OR extract only what the handler needs from userData.

b) cache entries are prevented by:
   → `cache` is in module/global scope → always a GC root
   → `cache` object holds references to all its property values
   → Those values never removed from `cache` object
   Fix: use LRU cache with max size, or WeakMap (if keys are objects).

c) `component` is prevented by:
   → window._debug.component still points to the original HeavyComponent
   → setting `component = null` only clears the local variable
   → window._debug.component still holds the strong reference
   Fix: window._debug.component = null (or delete window._debug.component).
```

</details>

---

### Exercise 2 — Memory-safe event system

Rewrite this event system to avoid memory leaks:

```javascript
// ❌ Leaky event system
class EventSystem {
  constructor() {
    this.listeners = {};
  }

  on(event, handler) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(handler);
  }

  emit(event, data) {
    (this.listeners[event] || []).forEach((h) => h(data));
  }
}

const events = new EventSystem();

// Components add listeners but never clean up
class Widget {
  constructor(name) {
    this.name = name;
    this.data = new Array(10_000);
    events.on("update", (d) => this.handleUpdate(d));
    // When Widget is "destroyed", listener stays forever
  }
}
```

<details>
<summary>Solution</summary>

```javascript
// ✅ Event system that returns unsubscribe functions
class EventSystem {
  #listeners = new Map();

  on(event, handler) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(handler);

    // Return unsubscribe function
    return () => {
      this.#listeners.get(event)?.delete(handler);
    };
  }

  emit(event, data) {
    this.#listeners.get(event)?.forEach((h) => h(data));
  }

  removeAllListeners(event) {
    if (event) {
      this.#listeners.delete(event);
    } else {
      this.#listeners.clear();
    }
  }
}

const events = new EventSystem();

class Widget {
  constructor(name) {
    this.name = name;
    this.data = new Array(10_000);

    // Store unsubscribe function
    this._unsubscribe = events.on("update", (d) => this.handleUpdate(d));
  }

  handleUpdate(data) {
    // use data
  }

  destroy() {
    this._unsubscribe(); // remove listener → allows GC
    this._unsubscribe = null;
    this.data = null;
  }
}

const w = new Widget("chart");
// Later:
w.destroy(); // properly cleans up
```

</details>

---

### Exercise 3 — TypedArray vs Array benchmark

Run this in your browser console and observe the memory difference:

```javascript
// Test 1: Regular Array
const regularStart = performance.memory?.usedJSHeapSize ?? 0;
const regular = new Array(500_000);
for (let i = 0; i < regular.length; i++) regular[i] = i * 1.5;
const regularUsed = (performance.memory?.usedJSHeapSize ?? 0) - regularStart;
console.log("Regular Array:", (regularUsed / 1024 / 1024).toFixed(2) + "MB");

// Test 2: Float64Array
const typedStart = performance.memory?.usedJSHeapSize ?? 0;
const typed = new Float64Array(500_000);
for (let i = 0; i < typed.length; i++) typed[i] = i * 1.5;
const typedUsed = (performance.memory?.usedJSHeapSize ?? 0) - typedStart;
console.log("Float64Array:", (typedUsed / 1024 / 1024).toFixed(2) + "MB");
```

Expected results:

- Regular Array: ~4–12MB (numbers boxed as objects + array overhead)
- Float64Array: ~4MB exactly (500,000 × 8 bytes, no overhead)

---

## 🔗 Related Topics

- [`javascript-core/09-garbage-collection.md`](./09-garbage-collection.md) — GC algorithms in depth
- [`performance/05-memory-leaks.md`](../performance/05-memory-leaks.md) — Finding and fixing leaks
- [`debugging/02-memory-tab.md`](../debugging/02-memory-tab.md) — DevTools Memory tab mastery
- [`javascript-core/05-closures.md`](./05-closures.md) — How closures retain memory

---

<div align="center">

**Next:** [`javascript-core/09-garbage-collection.md`](./09-garbage-collection.md) →

</div>
