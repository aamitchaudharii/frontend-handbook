# 05 — Memory Leaks

> **"A memory leak is not when memory is used. It's when memory is used and never returned — even though the code that needed it is long gone."**

Memory leaks in frontend applications are insidious. They don't crash your app immediately. They slowly degrade it: pages get sluggish, animations start stuttering, tabs consume gigabytes of RAM until the browser kills them. In a long-running SPA, a small leak compounds over hours of use into a critical problem.

This document covers the mechanisms of memory leaks in JavaScript, the six most common patterns found in production codebases, how to detect them with DevTools, and how to architect code that doesn't leak.

---

## 📚 Table of Contents

1. [How JavaScript Memory Works](#1-how-javascript-memory-works)
2. [What Is a Memory Leak?](#2-what-is-a-memory-leak)
3. [Garbage Collection — V8 Internals](#3-garbage-collection--v8-internals)
4. [The 6 Most Common Leak Patterns](#4-the-6-most-common-leak-patterns)
5. [Detecting Leaks with Chrome DevTools](#5-detecting-leaks-with-chrome-devtools)
6. [Measuring Memory Programmatically](#6-measuring-memory-programmatically)
7. [Fixing Leaks — Patterns & Architecture](#7-fixing-leaks--patterns--architecture)
8. [Memory-Safe Component Design](#8-memory-safe-component-design)
9. [Good Practices](#9-good-practices)
10. [Bad Practices](#10-bad-practices)
11. [Real-World Case Studies](#11-real-world-case-studies)
12. [Interview-Level Explanation](#12-interview-level-explanation)
13. [Exercises](#13-exercises)

---

## 1. How JavaScript Memory Works

JavaScript manages memory automatically. You don't call `malloc()` or `free()`. The engine handles allocation and deallocation. Understanding _how_ it does this is the foundation for understanding why leaks happen.

### The Memory Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    JAVASCRIPT MEMORY LIFECYCLE                   │
│                                                                   │
│   1. ALLOCATE        2. USE              3. RELEASE              │
│                                                                   │
│   const obj = {}  →  obj.name = 'hi'  →  obj goes out of scope  │
│   (heap: +48 bytes)  (still in heap)     (heap: -48 bytes)       │
│                                                                   │
│   JavaScript automates step 3 via Garbage Collection            │
│   A LEAK is when step 3 never happens for objects you're done with│
└─────────────────────────────────────────────────────────────────┘
```

### The Heap

All JavaScript objects, strings, closures, and arrays live in the **heap** — a region of memory managed by V8.

```javascript
// Each of these allocates on the heap:
const obj = { name: "Alice" }; // ~200 bytes
const arr = new Array(1000); // ~8,000 bytes
const str = "x".repeat(1_000_000); // ~2 MB
const fn = () => ({ captured: obj }); // closure — holds reference to obj
const el = document.createElement("div"); // DOM node — heap + native memory
```

### Reachability — The Core Concept

V8's garbage collector doesn't track "who's using this." It tracks **reachability**: can this object be reached by traversing references from a **root**?

**Roots are:**

- Global variables (`window`, `globalThis`)
- The current call stack (local variables in executing functions)
- Active DOM nodes (nodes in the document)
- Objects held by native code (event listeners registered with the browser)

```
REACHABILITY GRAPH:

window (root)
  └── app
        └── userService
              └── cache
                    └── userData ← REACHABLE → kept alive by GC

(no path from any root)
  └── oldComponent
        └── bigData ← UNREACHABLE → eligible for GC
```

**A memory leak is an object that is reachable (so GC can't collect it) but will never be used again.**

The GC doesn't know your intentions. It only knows the reference graph. If a reference exists, the object stays alive.

---

## 2. What Is a Memory Leak?

A memory leak in JavaScript is when:

1. You allocate memory (create an object, bind an event listener, set a timer)
2. You stop needing it (a component unmounts, a user navigates away)
3. **But a reference still exists somewhere** (a closure, a global, a detached DOM node)
4. The GC cannot collect it
5. This repeats over time → memory grows without bound

### What Leaks Look Like in Practice

```
Memory usage over time:

Normal app (GC working):
MB │    ╭╮  ╭╮  ╭╮   GC runs ↓  ╭╮
   │   ╭╯╰╮╭╯╰╮╭╯╰╮             ╭╯╰╮
   │───╯   ╰╯   ╰╯  ╰──────────╯    ╰──
   └────────────────────────────────────▶ time
   (sawtooth pattern: allocate, GC collects, allocate...)

Leaking app:
MB │                                    ╭──
   │                               ╭───╯
   │                          ╭───╯
   │                     ╭───╯
   │               ╭────╯
   │        ╭─────╯
   │───────╯
   └────────────────────────────────────▶ time
   (steady climb: GC runs but can't collect — references still held)
```

The sawtooth is healthy. The steady climb is a leak.

---

## 3. Garbage Collection — V8 Internals

V8 uses a **generational garbage collector** with two main regions:

### The Young Generation (Nursery)

- **Where:** Most new objects are allocated here
- **Size:** ~1–8 MB (small, intentionally)
- **Collection:** Very frequent, very fast (~1ms) — called **Scavenge**
- **Algorithm:** Copying collector — live objects copied to a new space, dead objects abandoned

```
Young Gen (before Scavenge):
┌─────────────────────────────┐
│ [obj A] [obj B] [obj C] ... │  ← new allocations (mixed live/dead)
└─────────────────────────────┘

Young Gen (after Scavenge):
┌─────────────────────────────┐
│ [obj A] [obj C]             │  ← only live objects remain (compacted)
└─────────────────────────────┘    obj B was unreachable → collected
```

Objects that survive multiple Scavenges are **promoted** to the old generation.

### The Old Generation (Tenured)

- **Where:** Long-lived objects, large allocations
- **Size:** Much larger (up to GBs)
- **Collection:** Less frequent, more expensive — called **Major GC** (Mark-Sweep-Compact)
- **Algorithm:** Mark all reachable objects, sweep unreachable, compact gaps

```
Mark phase:    Traverse reference graph from roots, mark all reachable objects
Sweep phase:   Scan heap, reclaim memory of unmarked objects
Compact phase: Move live objects together to reduce fragmentation
```

**Why this matters for leaks:**

- Short-lived objects (temporary variables in functions) are cheap — they die young in the nursery
- **Leaked objects always end up in the old generation** — they keep surviving Scavenges because references exist, get promoted, and stay there forever
- Major GC is expensive and can cause **GC pauses** — visible as frozen frames in your app

### GC Pauses

```
Normal frame: [JS 4ms] [Layout 2ms] [Paint 2ms] = 8ms ✅

Frame with GC pause:
[JS 4ms] [GC 80ms!!] [Layout 2ms] [Paint 2ms] = 88ms ❌
                ↑
     Page freezes for 80ms — user sees jank
```

V8 has moved to **incremental marking** and **concurrent sweeping** to reduce pause times, but large heaps still cause noticeable pauses. The best solution: don't accumulate a large heap.

---

## 4. The 6 Most Common Leak Patterns

### Pattern 1 — Forgotten Event Listeners

The most common leak in SPAs. An event listener keeps a reference to its callback — and the callback keeps a reference to everything it closes over.

```javascript
// LEAKS: listener added but never removed
class SearchComponent {
  constructor() {
    this.results = new Array(10000).fill({ data: "large payload" }); // 10k items

    // This listener closes over `this` — which includes this.results
    // Even when SearchComponent is "destroyed", the window still holds
    // a reference to this closure → entire SearchComponent + results leak
    window.addEventListener("keydown", (e) => {
      if (e.key === "Escape") this.close();
    });
  }

  close() {
    /* unmount logic */
  }
}

// Each time user navigates to the search page and back:
// new SearchComponent() → new listener → new 10k items in memory
// Old ones never collected — leak compounds on every navigation
```

```javascript
// FIXED: track and remove listeners on cleanup
class SearchComponent {
  constructor() {
    this.results = new Array(10000).fill({ data: "large payload" });

    // Bind once so we have a reference to remove later
    this._onKeydown = (e) => {
      if (e.key === "Escape") this.close();
    };
    window.addEventListener("keydown", this._onKeydown);
  }

  destroy() {
    // Remove listener — GC can now collect this component
    window.removeEventListener("keydown", this._onKeydown);
    this._onKeydown = null;
    this.results = null;
  }
}
```

**The rule:** Every `addEventListener` must have a corresponding `removeEventListener` when the component is unmounted/destroyed.

---

### Pattern 2 — Timers Not Cleared

`setInterval` and long `setTimeout` chains keep their callbacks alive — and everything those callbacks reference.

```javascript
// LEAKS: interval keeps running after component is gone
class LiveChart {
  constructor(container) {
    this.data = [];
    this.container = container;

    // This interval fires every second, forever
    // Even after LiveChart is "destroyed", the interval still has a
    // reference to `this` via the closure
    this.intervalId = setInterval(() => {
      this.data.push(fetchLatestPoint()); // keeps this.data growing forever
      this.render();
    }, 1000);
  }

  // User navigates away, component "destroyed" by the app
  // But nobody called destroy() → interval still firing
}
```

```javascript
// FIXED: always clear timers in cleanup
class LiveChart {
  constructor(container) {
    this.data = [];
    this.container = container;
    this.intervalId = setInterval(() => {
      this.data.push(fetchLatestPoint());
      this.render();
    }, 1000);
  }

  destroy() {
    clearInterval(this.intervalId);
    this.intervalId = null;
    this.data = null;
    this.container = null;
  }
}
```

**The rule:** Every `setInterval` must be cleared with `clearInterval`. Every `setTimeout` that might not have fired when cleanup happens must be cleared with `clearTimeout`.

---

### Pattern 3 — Detached DOM Nodes

A DOM node is **detached** when it's been removed from the document but JavaScript still holds a reference to it. The node (and its entire subtree) cannot be garbage collected.

```javascript
// LEAKS: reference held to removed node
class Modal {
  constructor() {
    // Create the modal DOM
    this.overlay = document.createElement("div");
    this.overlay.className = "modal-overlay";
    this.overlay.innerHTML = `
      <div class="modal-content">
        <button class="close-btn">Close</button>
        <!-- heavy content, images, nested elements -->
      </div>
    `;
    document.body.appendChild(this.overlay);
  }

  close() {
    document.body.removeChild(this.overlay);
    // ❌ this.overlay still referenced by this Modal instance
    // The entire DOM subtree is now DETACHED but not collected
    // If Modal instance stays alive (e.g., in an array of modals),
    // all that DOM memory leaks
  }
}

// Common version: cached DOM references to removed elements
const cache = {};
function cacheElement(id) {
  cache[id] = document.getElementById(id); // stored in global cache
}
// Later:
document.getElementById("tooltip").remove(); // removed from DOM
// cache[id] still holds reference → detached node → leak
```

```javascript
// FIXED: null out references to removed nodes
class Modal {
  constructor() {
    this.overlay = document.createElement("div");
    document.body.appendChild(this.overlay);
  }

  close() {
    document.body.removeChild(this.overlay);
    this.overlay = null; // allow GC to collect the entire subtree
  }

  destroy() {
    if (this.overlay && this.overlay.parentNode) {
      this.overlay.parentNode.removeChild(this.overlay);
    }
    this.overlay = null;
  }
}
```

**How to detect:** In DevTools Memory tab, a heap snapshot with "Detached HTMLDivElement" or "Detached HTMLElement" nodes indicates this leak pattern.

---

### Pattern 4 — Closures Capturing Large Scope

Closures keep their entire enclosing scope alive. A small callback that captures a large variable will prevent that variable from being collected.

```javascript
// LEAKS: closure captures large data unintentionally
function setupButton(buttonEl) {
  // This large dataset is in scope here
  const hugeDataset = fetchAllUserData(); // 50MB of data

  buttonEl.addEventListener("click", function handler() {
    // This handler only needs userId — but it closes over the entire
    // hugeDataset scope, keeping 50MB alive as long as this button exists
    const userId = hugeDataset[0].id;
    navigate(`/user/${userId}`);
  });
}
```

```javascript
// FIXED: extract only what you need before the closure
function setupButton(buttonEl) {
  const hugeDataset = fetchAllUserData();

  // Extract the specific value needed
  const userId = hugeDataset[0].id;

  // hugeDataset is no longer referenced in the closure
  // GC can collect it after setupButton returns

  buttonEl.addEventListener("click", function handler() {
    navigate(`/user/${userId}`); // only captures userId (a string)
  });
}
```

### Closure Leak in Event Handlers with Circular References

```javascript
// LEAKS: circular reference through closure
function attachHandler(element) {
  // element → handler (listener) → element (closure captures element)
  // This circular reference prevents collection in older GCs
  // Modern V8 handles cycles, but it's still bad practice

  element.addEventListener("click", function () {
    console.log(element.id); // closure captures `element`
  });
}

// FIXED: use event.currentTarget instead
element.addEventListener("click", function (event) {
  console.log(event.currentTarget.id); // no closure over element
});
```

---

### Pattern 5 — Growing Caches Without Eviction

Any in-memory cache that grows without a size limit or expiration policy will eventually consume all available memory.

```javascript
// LEAKS: unbounded cache
const apiCache = new Map(); // grows forever

async function fetchUser(id) {
  if (apiCache.has(id)) return apiCache.get(id);

  const user = await fetch(`/api/users/${id}`).then((r) => r.json());
  apiCache.set(id, user); // never removed
  return user;
}
// After millions of unique user IDs: apiCache has millions of entries in memory
```

```javascript
// FIXED: LRU cache with size limit
class LRUCache {
  constructor(maxSize = 100) {
    this.maxSize = maxSize;
    this.cache = new Map(); // Map preserves insertion order
  }

  get(key) {
    if (!this.cache.has(key)) return undefined;
    // Move to end (most recently used)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Evict least recently used (first entry in Map)
      const lruKey = this.cache.keys().next().value;
      this.cache.delete(lruKey);
    }
    this.cache.set(key, value);
  }

  has(key) {
    return this.cache.has(key);
  }
  delete(key) {
    return this.cache.delete(key);
  }
  clear() {
    this.cache.clear();
  }
  get size() {
    return this.cache.size;
  }
}

const apiCache = new LRUCache(200); // max 200 entries
```

**Alternative: Use `WeakMap` for object-keyed caches**

```javascript
// WeakMap: keys are held weakly — when the key object is GC'd,
// the entry is automatically removed. Zero manual eviction needed.
const elementDataCache = new WeakMap();

function setElementData(element, data) {
  elementDataCache.set(element, data); // key is a DOM element
}

function getElementData(element) {
  return elementDataCache.get(element);
}

// When the DOM element is removed and all JS references dropped,
// the WeakMap entry is automatically eligible for GC.
// No manual cleanup needed.
```

---

### Pattern 6 — Observer and Subscription Leaks

MutationObserver, IntersectionObserver, ResizeObserver, and custom pub-sub subscriptions all keep references alive until explicitly disconnected.

```javascript
// LEAKS: observer never disconnected
class InfiniteScrollComponent {
  constructor(container) {
    this.container = container;
    this.items = [];

    // This observer holds a reference to `this` via the callback
    this.observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting) {
        this.loadMoreItems(); // captures `this`
      }
    });

    this.sentinel = document.createElement("div");
    container.appendChild(this.sentinel);
    this.observer.observe(this.sentinel);
    // If component is destroyed without calling destroy(),
    // observer keeps this component alive
  }
}
```

```javascript
// FIXED: disconnect all observers in cleanup
class InfiniteScrollComponent {
  constructor(container) {
    this.container = container;
    this.items = [];

    this.observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting) this.loadMoreItems();
    });

    this.sentinel = document.createElement("div");
    container.appendChild(this.sentinel);
    this.observer.observe(this.sentinel);
  }

  destroy() {
    this.observer.disconnect(); // stop observing all targets
    this.observer = null;
    this.sentinel.remove();
    this.sentinel = null;
    this.container = null;
    this.items = null;
  }
}
```

### Pub-Sub Subscription Leaks

```javascript
// LEAKS: subscription never cancelled
class UserProfile {
  constructor(userId) {
    this.userId = userId;
    this.data = null;

    // Subscribes to global event bus
    // If UserProfile is "destroyed" without unsubscribing,
    // the event bus holds a reference → UserProfile can't be GC'd
    eventBus.on("userUpdated", (user) => {
      if (user.id === this.userId) {
        this.data = user;
        this.render();
      }
    });
  }
}
```

```javascript
// FIXED: store subscription token, unsubscribe on destroy
class UserProfile {
  constructor(userId) {
    this.userId = userId;
    this.data = null;

    const handler = (user) => {
      if (user.id === this.userId) {
        this.data = user;
        this.render();
      }
    };

    // Store the unsubscribe function
    this._unsubscribe = eventBus.on("userUpdated", handler);
  }

  destroy() {
    this._unsubscribe(); // remove from event bus
    this._unsubscribe = null;
    this.data = null;
  }
}

// Event bus should return an unsubscribe function:
class EventBus {
  constructor() {
    this._listeners = new Map();
  }

  on(event, handler) {
    if (!this._listeners.has(event)) this._listeners.set(event, new Set());
    this._listeners.get(event).add(handler);
    // Return unsubscribe function
    return () => this._listeners.get(event)?.delete(handler);
  }

  emit(event, data) {
    this._listeners.get(event)?.forEach((h) => h(data));
  }
}
```

---

## 5. Detecting Leaks with Chrome DevTools

### The Three-Snapshot Technique

This is the standard workflow for finding memory leaks:

```
Step 1: Open DevTools → Memory tab
Step 2: Select "Heap snapshot"
Step 3: Click "Take snapshot" → Snapshot 1 (baseline)
Step 4: Perform the action you suspect leaks (navigate to page, open modal, etc.)
Step 5: Click "Take snapshot" → Snapshot 2 (after action)
Step 6: Perform the action again (to confirm the pattern)
Step 7: Click "Take snapshot" → Snapshot 3 (second repeat)

Step 8: Select Snapshot 3
Step 9: Change view from "Summary" to "Comparison"
Step 10: Compare against Snapshot 1

Objects with positive "# New" that are suspicious = your leak candidates
```

### Reading Heap Snapshots

```
Heap Snapshot view columns:

Constructor          — Object type/class
Distance             — Steps from GC root (closer = more likely root cause)
Shallow Size         — Memory this object directly uses
Retained Size        — Memory that would be freed if this object was collected
                       (this is the important number for leaks)

Red entries = detached DOM nodes (almost always a leak)
```

**What to look for:**

- Detached HTMLElement, HTMLDivElement, etc. → Pattern 3 (detached DOM)
- Growing arrays of identical objects → Pattern 5 (unbounded cache)
- Closures with large retained sizes → Pattern 4 (closure scope capture)
- EventListener or EventTarget counts growing → Pattern 1 (forgotten listeners)

### The Allocation Timeline

For leaks that happen continuously (not just on an action):

```
Memory tab → Allocation instrumentation on timeline → Start
  → Use the app for 30 seconds
  → Stop
  → Look for blue bars that don't get collected (grey) over time
  → Click a blue bar → see what was allocated and not freed
```

### The Performance Monitor

For a live view of memory growth:

```
DevTools → ⋮ → More tools → Performance monitor
  → Watch "JS heap size" while using the app
  → Healthy: rises then falls as GC runs (sawtooth)
  → Leaking: steady climb with GC not reducing it
```

### Checking for Detached DOM Nodes

In the Console tab:

```javascript
// Quick check for detached DOM nodes
const snapshot = performance.memory; // basic stats
console.log(
  "Heap used:",
  (snapshot.usedJSHeapSize / 1024 / 1024).toFixed(1) + " MB",
);
console.log(
  "Heap limit:",
  (snapshot.jsHeapSizeLimit / 1024 / 1024).toFixed(1) + " MB",
);
```

In Heap Snapshot, filter by "Detached":

```
Heap Snapshot → Class filter → type "Detached"
  → Lists all detached DOM nodes still in memory
  → Click any → see "Retainers" panel
  → Retainers show WHY the node is still alive (which reference holds it)
```

---

## 6. Measuring Memory Programmatically

### `performance.memory` (Chrome only)

```javascript
function logMemory(label = "") {
  if (!performance.memory) return; // only in Chrome

  const mb = (bytes) => (bytes / 1024 / 1024).toFixed(2) + " MB";

  console.log(
    `[Memory${label ? ": " + label : ""}]`,
    "Used:",
    mb(performance.memory.usedJSHeapSize),
    "/ Total:",
    mb(performance.memory.totalJSHeapSize),
    "/ Limit:",
    mb(performance.memory.jsHeapSizeLimit),
  );
}

// Usage: bracket the suspicious operation
logMemory("before");
await navigateToHeavyPage();
logMemory("after navigation");
await navigateBack();
logMemory("after back");
// If "after back" is significantly higher than "before", you have a leak
```

### PerformanceObserver for Memory

```javascript
// Observe memory-related performance entries
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === "measure") {
      console.log(entry.name, entry.duration.toFixed(1) + "ms");
    }
  }
});
observer.observe({ entryTypes: ["measure"] });
```

### Automated Leak Detection Test

```javascript
// Test for leaks by measuring heap before and after
async function testForLeak(actionFn, options = {}) {
  const { iterations = 5, gcBefore = true } = options;

  // Force GC if possible (requires --expose-gc Node flag or Chrome with flag)
  if (gcBefore && window.gc) window.gc();

  const before = performance.memory?.usedJSHeapSize ?? 0;

  for (let i = 0; i < iterations; i++) {
    await actionFn();
  }

  if (gcBefore && window.gc) window.gc();

  // Wait for GC to complete
  await new Promise((r) => setTimeout(r, 100));

  const after = performance.memory?.usedJSHeapSize ?? 0;
  const diff = after - before;
  const leaked = diff > 1024 * 1024; // > 1MB after iterations = suspicious

  console.log(
    `Memory change after ${iterations} iterations: ${(diff / 1024).toFixed(
      1,
    )} KB ${leaked ? "⚠️ POSSIBLE LEAK" : "✅ OK"}`,
  );

  return { before, after, diff, leaked };
}

// Usage:
await testForLeak(async () => {
  const modal = new Modal();
  modal.open();
  await wait(100);
  modal.close();
  modal.destroy(); // test that destroy() actually cleans up
});
```

---

## 7. Fixing Leaks — Patterns & Architecture

### The Cleanup Contract

Every component/class that:

- Registers event listeners
- Creates timers
- Subscribes to observables/event buses
- Creates observers (MutationObserver, IntersectionObserver, etc.)
- Holds large data structures
- References DOM elements

...must implement a `destroy()` or `cleanup()` method that reverses all of it.

```javascript
// The cleanup contract — every class follows this interface
class ComponentBase {
  constructor() {
    this._cleanupFns = []; // collect all cleanup callbacks
  }

  /** Register a cleanup function to run when component is destroyed */
  _onCleanup(fn) {
    this._cleanupFns.push(fn);
  }

  /** Call this when component is removed */
  destroy() {
    this._cleanupFns.forEach((fn) => fn());
    this._cleanupFns = [];
  }
}

// Usage:
class SearchBar extends ComponentBase {
  constructor(container) {
    super();

    this.input = container.querySelector("input");
    this.results = [];

    // Register handler
    const onInput = (e) => this.handleInput(e);
    this.input.addEventListener("input", onInput);
    this._onCleanup(() => this.input.removeEventListener("input", onInput));

    // Register timer
    this.debounceTimer = null;
    this._onCleanup(() => clearTimeout(this.debounceTimer));

    // Register observer
    const ro = new ResizeObserver(() => this.handleResize());
    ro.observe(this.input);
    this._onCleanup(() => ro.disconnect());

    // Register subscription
    const unsub = searchStore.subscribe((s) => (this.results = s.results));
    this._onCleanup(unsub);
  }

  // destroy() inherited — calls all registered cleanup functions
}
```

### AbortController for Async Cleanup

```javascript
// AbortController lets you cancel fetch requests and event listeners
class DataLoader {
  constructor() {
    this.controller = new AbortController();
  }

  async loadData(url) {
    try {
      const response = await fetch(url, {
        signal: this.controller.signal, // fetch is cancelled when abort() is called
      });
      return response.json();
    } catch (e) {
      if (e.name === "AbortError") return null; // cancelled — not an error
      throw e;
    }
  }

  // Use AbortController for event listeners too (modern browsers)
  setupListeners(element) {
    const { signal } = this.controller;
    element.addEventListener("click", this.handleClick, { signal });
    element.addEventListener("mouseover", this.handleHover, { signal });
    // When abort() is called, ALL listeners with this signal are removed
    // No need to track each one individually
  }

  destroy() {
    this.controller.abort(); // cancels fetches AND removes all event listeners
  }
}
```

### WeakRef for Optional References

When you want to hold a reference to an object without preventing its collection:

```javascript
// WeakRef: holds reference without preventing GC
class ComponentRegistry {
  constructor() {
    this._registry = new Map(); // id → WeakRef<Component>
    this._finalizer = new FinalizationRegistry((id) => {
      // Called when a component is GC'd
      this._registry.delete(id);
      console.log(`Component ${id} was garbage collected`);
    });
  }

  register(id, component) {
    this._registry.set(id, new WeakRef(component));
    this._finalizer.register(component, id);
  }

  get(id) {
    const ref = this._registry.get(id);
    if (!ref) return undefined;

    const component = ref.deref(); // deref returns undefined if GC'd
    if (!component) {
      this._registry.delete(id);
      return undefined;
    }
    return component;
  }
}
```

---

## 8. Memory-Safe Component Design

### The Lifecycle Pattern

```javascript
/**
 * Memory-safe vanilla JS component base
 * Every component follows create → mount → unmount lifecycle
 */
class Component {
  constructor(props = {}) {
    this.props = props;
    this.el = null; // root DOM element
    this._subs = []; // subscriptions
    this._timers = []; // timer IDs
    this._obs = []; // observers
    this._listeners = []; // [element, event, handler] tuples
  }

  // ── Lifecycle ────────────────────────────────────────────────────

  mount(container) {
    this.el = this.render();
    container.appendChild(this.el);
    this.afterMount();
    return this;
  }

  unmount() {
    this.beforeUnmount();
    this._cleanup();
    if (this.el?.parentNode) this.el.parentNode.removeChild(this.el);
    this.el = null;
  }

  afterMount() {} // override in subclass
  beforeUnmount() {} // override in subclass
  render() {
    return document.createElement("div");
  } // override

  // ── Safe resource registration ────────────────────────────────────

  _addEventListener(el, event, handler, options) {
    el.addEventListener(event, handler, options);
    this._listeners.push([el, event, handler, options]);
  }

  _setTimeout(fn, delay) {
    const id = setTimeout(fn, delay);
    this._timers.push({ type: "timeout", id });
    return id;
  }

  _setInterval(fn, delay) {
    const id = setInterval(fn, delay);
    this._timers.push({ type: "interval", id });
    return id;
  }

  _observe(observer) {
    this._obs.push(observer);
    return observer;
  }

  _subscribe(unsubFn) {
    this._subs.push(unsubFn);
    return unsubFn;
  }

  // ── Cleanup ────────────────────────────────────────────────────────

  _cleanup() {
    // Remove event listeners
    this._listeners.forEach(([el, evt, handler, opts]) => {
      el.removeEventListener(evt, handler, opts);
    });
    this._listeners = [];

    // Clear timers
    this._timers.forEach(({ type, id }) => {
      type === "interval" ? clearInterval(id) : clearTimeout(id);
    });
    this._timers = [];

    // Disconnect observers
    this._obs.forEach((obs) => obs.disconnect?.() ?? obs.unobserve?.());
    this._obs = [];

    // Cancel subscriptions
    this._subs.forEach((unsub) => unsub());
    this._subs = [];
  }
}

// Usage:
class LiveCounter extends Component {
  constructor(props) {
    super(props);
    this.count = 0;
  }

  render() {
    const el = document.createElement("div");
    el.className = "counter";
    el.textContent = this.count;
    this.counterEl = el;
    return el;
  }

  afterMount() {
    // Safe: automatically cleaned up on unmount
    this._setInterval(() => {
      this.count++;
      this.counterEl.textContent = this.count;
    }, 1000);

    this._addEventListener(document, "keydown", (e) => {
      if (e.key === "r") this.count = 0;
    });
  }
  // No need to manually clean up — Component._cleanup() handles it
}
```

---

## 9. Good Practices

### ✅ Use `AbortController` for fetch + listener cleanup

```javascript
const ac = new AbortController();

// Both fetch and listener share the same signal
fetch("/api/data", { signal: ac.signal });
element.addEventListener("click", handler, { signal: ac.signal });

// One call cancels everything
function cleanup() {
  ac.abort();
}
```

### ✅ Use `WeakMap` and `WeakSet` for metadata attached to objects

```javascript
// WeakMap: automatically cleaned up when DOM element is GC'd
const elementMeta = new WeakMap();

function attachMeta(element, data) {
  elementMeta.set(element, data);
}
// No cleanup needed — when element is GC'd, entry is removed automatically
```

### ✅ Null out large references on cleanup

```javascript
destroy() {
  this.largeDataset = null;  // allow GC immediately
  this.cachedElements = null;
  this.imageBuffer = null;
}
```

### ✅ Use `once` option for single-fire listeners

```javascript
// Automatically removed after firing once — no manual cleanup
element.addEventListener("animationend", handler, { once: true });
```

### ✅ Bound methods vs arrow functions in classes

```javascript
class Component {
  constructor() {
    // Arrow function in class field — same reference, removable
    this.handleClick = (e) => this.onClick(e);
    element.addEventListener("click", this.handleClick);
  }
  destroy() {
    element.removeEventListener("click", this.handleClick); // works: same reference
  }
}

// vs. anonymous arrow in addEventListener — NOT removable:
element.addEventListener("click", (e) => this.onClick(e)); // new fn every time
element.removeEventListener("click", (e) => this.onClick(e)); // different reference — does nothing
```

---

## 10. Bad Practices

### ❌ Anonymous listeners (impossible to remove)

```javascript
// Can never be removed — reference lost immediately
element.addEventListener("click", function () {
  doSomething(); // this listener lives forever
});
```

### ❌ Global variable accumulation

```javascript
// Classic leak: pushing to a global array forever
window.debugLog = window.debugLog || [];
function log(msg) {
  window.debugLog.push({ msg, time: Date.now(), stack: new Error().stack });
  // In production: debugLog grows without bound
}
```

### ❌ Storing DOM references in module-level variables

```javascript
// Module-level: lives for the entire app lifetime
const elementCache = {};

function cacheEl(id) {
  elementCache[id] = document.getElementById(id);
}
// Elements removed from DOM still referenced here — never GC'd
```

### ❌ Console.log in production keeping references

```javascript
// console.log keeps objects alive in DevTools memory panel
// (the DevTools Console holds a reference so you can inspect later)
console.log(largeObject); // if DevTools is open, this object leaks in DevTools
// Not a production leak per se, but confuses memory profiling
```

### ❌ Closures in loops creating N closures

```javascript
// Creates N separate closures, each with its own scope
for (let i = 0; i < elements.length; i++) {
  elements[i].addEventListener("click", function () {
    console.log(i); // each closure captures a different `i`
    // All N closures held alive by the listeners
  });
}
// Fix: use event delegation — one listener on parent
```

---

## 11. Real-World Case Studies

### Case Study 1 — SPA Navigation Leak

**Symptom:** Tab memory grew from 50MB on page load to 800MB after 30 minutes of use. Users reported slowdowns after extended sessions.

**Investigation:** Heap snapshot comparison after 10 page navigations showed thousands of `EventListener` objects, all pointing to component callbacks.

**Root cause:**

```javascript
// ❌ Component framework not cleaning up on route change
class PageComponent {
  mount() {
    window.addEventListener("resize", this.handleResize.bind(this));
    document.addEventListener("keydown", this.handleKeydown.bind(this));
    store.subscribe(this.handleStoreUpdate.bind(this));
  }
  // No unmount/destroy method → all listeners accumulate
}
// After 30 navigations: 30 resize handlers, 30 keydown handlers, 30 store subs
// All still active, all holding page component instances (and their data) alive
```

**Fix:** Added router lifecycle hooks that called `destroy()` on outgoing page components. Listeners and subscriptions cleaned up on navigation.

**Result:** Memory stabilized at ~55MB regardless of navigation count.

---

### Case Study 2 — Image Viewer Memory Explosion

**Symptom:** An image gallery component consumed 1.5GB of RAM after viewing 50 images.

**Root cause:**

```javascript
// ❌ Keeping ALL decoded image data in memory
class ImageGallery {
  constructor() {
    this.imageCache = {}; // never evicted
  }

  async loadImage(url) {
    if (this.imageCache[url]) return this.imageCache[url];

    const blob = await fetch(url).then((r) => r.blob());
    const bitmap = await createImageBitmap(blob); // decoded: ~30MB per image
    this.imageCache[url] = bitmap; // stored forever
    return bitmap;
  }
}
// 50 images × 30MB decoded = 1.5GB
```

**Fix:**

```javascript
// ✅ LRU cache — keep only last 5 images decoded
class ImageGallery {
  constructor() {
    this.imageCache = new LRUCache(5); // max 5 × 30MB = 150MB max
  }

  async loadImage(url) {
    let bitmap = this.imageCache.get(url);
    if (!bitmap) {
      const blob = await fetch(url).then((r) => r.blob());
      bitmap = await createImageBitmap(blob);
      this.imageCache.set(url, bitmap);
    }
    return bitmap;
  }
}
```

**Result:** Memory stayed under 200MB regardless of images viewed.

---

### Case Study 3 — WebSocket Subscription Leak

**Symptom:** Real-time dashboard used more and more CPU over time, eventually making the tab unresponsive.

**Root cause:**

```javascript
// ❌ New subscription on every data refresh, old ones never removed
class DataWidget {
  initialize() {
    this.socket = getWebSocket();
    // Called every time data format changed — multiple times per session
    this.socket.on("data", (payload) => {
      this.processData(payload); // each call registers a NEW handler
      // Old handler still registered — receives ALL future messages too
    });
  }
}
// After 100 re-initializations: 100 handlers fire on every WebSocket message
// CPU usage: 100× what it should be
```

**Fix:**

```javascript
// ✅ Track and remove handler before re-registering
class DataWidget {
  initialize() {
    if (this._dataHandler) {
      this.socket.off("data", this._dataHandler); // remove old
    }
    this._dataHandler = (payload) => this.processData(payload);
    this.socket.on("data", this._dataHandler); // register new
  }

  destroy() {
    if (this._dataHandler) {
      this.socket.off("data", this._dataHandler);
      this._dataHandler = null;
    }
  }
}
```

---

## 12. Interview-Level Explanation

> **"What causes memory leaks in JavaScript and how do you find and fix them?"**

**Strong answer:**

> "Memory leaks in JavaScript happen when objects are allocated and then never released — not because the GC is broken, but because references still exist somewhere in the code, keeping objects reachable from a root.
>
> The most common patterns are: forgotten event listeners — where a component adds a window or document listener but never removes it when it's unmounted, so the component and everything it references stays alive; uncleared timers — setInterval callbacks hold closures that keep component instances alive; detached DOM nodes — elements removed from the document but still referenced in JavaScript variables; unbounded caches — Maps or arrays that grow indefinitely; and observer leaks — IntersectionObserver, MutationObserver, or custom pub-sub subscriptions that are never disconnected.
>
> To find them, I use the three-snapshot technique in Chrome DevTools Memory tab: take a heap snapshot before an action, perform the action multiple times, take another snapshot, then compare. Objects with growing retained sizes that shouldn't grow — especially detached DOM nodes or component instances — point to the leak.
>
> The fix is architectural: every component that registers resources must implement a destroy method that reverses all registrations. Using AbortController unifies cancellation of both fetch requests and event listeners. For caches, using LRU with a size limit or WeakMap for object-keyed data prevents unbounded growth. And null-ing out large references in cleanup allows GC to collect them immediately rather than waiting for the GC to trace the entire heap."

---

## 13. Exercises

### Exercise 1 — Find the leak

```javascript
class NotificationBanner {
  constructor() {
    this.messages = [];
    this.el = document.createElement("div");
    document.body.appendChild(this.el);

    window.addEventListener("online", () => {
      this.showMessage("Back online!");
    });

    window.addEventListener("offline", () => {
      this.showMessage("Connection lost");
    });

    document.addEventListener("visibilitychange", () => {
      if (document.hidden) this.pause();
      else this.resume();
    });

    this.pollInterval = setInterval(() => {
      this.checkForNewMessages();
    }, 5000);
  }

  showMessage(text) {
    this.messages.push({ text, time: Date.now() });
    this.render();
  }

  destroy() {
    document.body.removeChild(this.el);
  }
}
```

<details>
<summary>Identified leaks + fix</summary>

**Leaks:**

1. `window` 'online' listener — never removed
2. `window` 'offline' listener — never removed
3. `document` 'visibilitychange' listener — never removed
4. `setInterval` — never cleared
5. `this.messages` — grows unboundedly (unbounded cache)
6. `this.el` still referenced after `removeChild` (minor — nulling out is good practice)

**Fixed version:**

```javascript
class NotificationBanner {
  constructor() {
    this.messages = [];
    this.el = document.createElement("div");
    document.body.appendChild(this.el);

    this._onOnline = () => this.showMessage("Back online!");
    this._onOffline = () => this.showMessage("Connection lost");
    this._onVisChange = () => (document.hidden ? this.pause() : this.resume());

    window.addEventListener("online", this._onOnline);
    window.addEventListener("offline", this._onOffline);
    document.addEventListener("visibilitychange", this._onVisChange);

    this.pollInterval = setInterval(() => this.checkForNewMessages(), 5000);
  }

  showMessage(text) {
    this.messages.push({ text, time: Date.now() });
    if (this.messages.length > 50) this.messages.shift(); // cap at 50
    this.render();
  }

  destroy() {
    window.removeEventListener("online", this._onOnline);
    window.removeEventListener("offline", this._onOffline);
    document.removeEventListener("visibilitychange", this._onVisChange);
    clearInterval(this.pollInterval);

    if (this.el.parentNode) this.el.parentNode.removeChild(this.el);
    this.el = null;
    this.messages = null;
  }
}
```

</details>

---

### Exercise 2 — Profile a leak in DevTools

```javascript
// Paste this in DevTools Console
// It simulates a leaking component

class LeakyComponent {
  constructor(id) {
    this.id = id;
    this.data = new Array(10000).fill(`data-${id}`);
    window.addEventListener("resize", () => {
      console.log(`${this.id} handling resize`);
    });
  }
}

const components = [];

// Create and "destroy" 20 components
for (let i = 0; i < 20; i++) {
  const c = new LeakyComponent(`component-${i}`);
  components.push(c);
}

components.length = 0; // "clear" the array — but leak remains
```

1. Open Memory tab → Take heap snapshot (baseline)
2. Run the code above
3. Take another heap snapshot
4. Compare: look for LeakyComponent instances and growing arrays
5. Fix the code and verify the leak is gone

---

### Exercise 3 — Implement an LRU cache

Implement a production-ready LRU cache with:

- `get(key)` — O(1), moves key to most-recently-used
- `set(key, value)` — O(1), evicts LRU if at capacity
- `delete(key)` — O(1)
- `has(key)` — O(1)
- TTL support — entries expire after a configurable time

<details>
<summary>Solution</summary>

```javascript
class LRUCache {
  constructor({ maxSize = 100, ttl = Infinity } = {}) {
    this.maxSize = maxSize;
    this.ttl = ttl;
    this._cache = new Map(); // key → { value, expires }
  }

  get(key) {
    if (!this._cache.has(key)) return undefined;

    const entry = this._cache.get(key);

    // Check TTL
    if (Date.now() > entry.expires) {
      this._cache.delete(key);
      return undefined;
    }

    // Move to end (most recently used)
    this._cache.delete(key);
    this._cache.set(key, entry);
    return entry.value;
  }

  set(key, value) {
    // Remove if exists (to update position)
    if (this._cache.has(key)) this._cache.delete(key);

    // Evict LRU if at capacity
    if (this._cache.size >= this.maxSize) {
      const lruKey = this._cache.keys().next().value;
      this._cache.delete(lruKey);
    }

    this._cache.set(key, {
      value,
      expires: Date.now() + this.ttl,
    });
  }

  has(key) {
    if (!this._cache.has(key)) return false;
    const entry = this._cache.get(key);
    if (Date.now() > entry.expires) {
      this._cache.delete(key);
      return false;
    }
    return true;
  }

  delete(key) {
    return this._cache.delete(key);
  }
  clear() {
    this._cache.clear();
  }
  get size() {
    return this._cache.size;
  }
}

// Usage:
const cache = new LRUCache({ maxSize: 100, ttl: 5 * 60 * 1000 }); // 5 min TTL
cache.set("user:42", userData);
const user = cache.get("user:42"); // undefined after TTL
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/08-memory-management.md`](../javascript-core/08-memory-management.md) — JS heap and allocation
- [`javascript-core/09-garbage-collection.md`](../javascript-core/09-garbage-collection.md) — GC algorithms deep dive
- [`debugging/02-memory-tab.md`](../debugging/02-memory-tab.md) — DevTools Memory tab mastery
- [`anti-patterns/02-memory-leak-patterns.md`](../anti-patterns/02-memory-leak-patterns.md) — Full anti-pattern catalog
- [`patterns/01-observer.md`](../patterns/01-observer.md) — Observer pattern with safe cleanup

---

<div align="center">

**Next:** [`system-design/06-event-delegation.md`](../system-design/06-event-delegation.md) →

</div>
