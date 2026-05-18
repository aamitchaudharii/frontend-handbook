# 📖 Glossary

> **A reference for every technical term used throughout this handbook.** Terms are grouped by domain. Each entry explains the concept in plain English, then gives the precise technical meaning, with cross-references to where it appears in the handbook.

---

## How to Use This Glossary

- Use **Ctrl+F / Cmd+F** to find a specific term
- Terms in **bold** within definitions are themselves defined in this glossary
- `→ See:` links point to the handbook file where the topic is covered in depth
- Terms are grouped by domain, not alphabetically — related concepts stay together

---

## 🧠 JavaScript Engine

### V8

Google's JavaScript engine used in Chrome and Node.js. Parses, compiles (via JIT), and executes JavaScript. V8 is responsible for the **call stack**, **heap**, **garbage collection**, and **hidden classes**. It does NOT provide `setTimeout`, `fetch`, or DOM APIs — those come from the **host environment**.

`→ See:` [`javascript-core/08-memory-management.md`](../javascript-core/08-memory-management.md)

---

### JIT Compilation (Just-In-Time)

A compilation strategy where code is compiled to machine code at runtime, not ahead of time. V8 starts by interpreting JavaScript, then identifies "hot" code paths and compiles them to optimized machine code. If assumptions break (e.g., an object changes shape), V8 **deoptimizes** back to interpreted execution.

---

### Execution Context

The environment created by the JavaScript engine each time code is run. Contains three components: the **Variable Environment** (stores `var` declarations and function declarations), the **Lexical Environment** (stores `let`/`const` and the outer scope reference), and the **ThisBinding** (what `this` refers to). Every function call creates a new execution context.

`→ See:` [`javascript-core/01-execution-context.md`](../javascript-core/01-execution-context.md)

---

### Call Stack

A **LIFO (Last In, First Out)** data structure that tracks the currently executing functions. Each function call pushes a **stack frame** onto the stack. When the function returns, its frame is popped. JavaScript is single-threaded — only the topmost frame executes at any time.

`→ See:` [`javascript-core/02-call-stack.md`](../javascript-core/02-call-stack.md)

---

### Stack Frame

A single entry on the **call stack** representing one function invocation. Contains local variables, parameters, a return address, and a pointer to the current **execution context**.

---

### Stack Overflow

Error thrown when the **call stack** exceeds its size limit (typically ~10,000 frames in V8). Caused by infinite recursion or excessively deep call chains. Throws `RangeError: Maximum call stack size exceeded`.

---

### Heap

The region of memory where JavaScript objects, arrays, functions, and strings are allocated. Unlike the **stack**, heap memory is not automatically freed when a function returns — it's managed by the **garbage collector**.

`→ See:` [`javascript-core/08-memory-management.md`](../javascript-core/08-memory-management.md)

---

### Hidden Class (Shape / Map)

V8's internal representation of an object's structure — which properties it has and in what order. Objects with the same hidden class share optimized compiled code. Adding properties in a different order than other instances of the same constructor creates a new hidden class, degrading performance.

`→ See:` [`javascript-core/09-garbage-collection.md`](../javascript-core/09-garbage-collection.md)

---

### Deoptimization

When V8 reverts a function from optimized machine code back to slower interpreted execution. Triggered when the engine's assumptions about types or object shapes are violated at runtime.

---

## 🔄 Event Loop & Async

### Event Loop

The mechanism that manages JavaScript's concurrency. Repeatedly checks: if the **call stack** is empty, dequeue one **macrotask**, run it, then drain the entire **microtask queue**, then allow a **rendering checkpoint**. This loop runs continuously as long as the page is alive.

`→ See:` [`javascript-core/03-event-loop.md`](../javascript-core/03-event-loop.md)

---

### Macrotask (Task)

A unit of work processed by the **event loop** one at a time. The browser may render between macrotasks. Sources: `setTimeout`, `setInterval`, DOM event callbacks, `MessageChannel`, I/O callbacks. Contrast with **microtask**.

`→ See:` [`javascript-core/04-microtask-vs-macrotask.md`](../javascript-core/04-microtask-vs-macrotask.md)

---

### Microtask

A high-priority deferred callback that runs immediately after the current **macrotask** completes — before any rendering or next macrotask. The entire microtask queue is drained before moving on. Sources: `Promise.then/catch/finally`, `queueMicrotask()`, `MutationObserver` callbacks, `async/await` continuations.

`→ See:` [`javascript-core/04-microtask-vs-macrotask.md`](../javascript-core/04-microtask-vs-macrotask.md)

---

### Microtask Checkpoint

The point at which the engine drains the entire **microtask queue**. Occurs after every **macrotask**, after script evaluation, and at other spec-defined points. Until the microtask queue is empty, no rendering occurs.

---

### Microtask Starvation

When an infinite or very long chain of **microtasks** prevents the browser from ever reaching the **rendering checkpoint**. The page freezes because the GC and renderer can never run. Example: `Promise.resolve().then(() => Promise.resolve().then(...))` recursively.

---

### Rendering Checkpoint

The point in the **event loop** between microtask drain and next macrotask where the browser may repaint the screen. `requestAnimationFrame` callbacks run here, before layout and paint.

`→ See:` [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md)

---

### requestAnimationFrame (rAF)

A browser API that schedules a callback to run at the next **rendering checkpoint** — synced to the display's refresh rate (typically 60fps). Preferred over `setTimeout` for animations because it's synchronized with vsync and paused in background tabs.

`→ See:` [`performance/04-raf-optimization.md`](../performance/04-raf-optimization.md)

---

### requestIdleCallback (rIC)

A browser API that schedules a callback to run during **idle time** — when the browser has completed all rendering and input handling for the current frame and has spare time. Not guaranteed to run; use for non-critical background work.

---

### Cooperative Scheduling

Breaking long-running tasks into smaller chunks and yielding back to the **event loop** between chunks. Allows the browser to render frames and handle user input during long operations. Implemented using `setTimeout(0)`, `requestIdleCallback`, or the Scheduler API.

`→ See:` [`rendering/03-cooperative-scheduling.md`](../rendering/03-cooperative-scheduling.md)

---

### Long Task

Any **macrotask** that takes more than 50ms to complete. Long tasks cause perceptible jank because the browser cannot render or respond to input while the main thread is occupied. Detectable via the `PerformanceObserver` `longtask` entry type.

---

### Temporal Dead Zone (TDZ)

The period between the start of a **block scope** and the `let`/`const` declaration line. During the TDZ, the variable exists (it's been hoisted) but is not initialized. Accessing it throws `ReferenceError`. Prevents the silent `undefined` bugs caused by `var` hoisting.

`→ See:` [`javascript-core/07-scope-chain.md`](../javascript-core/07-scope-chain.md)

---

### Hoisting

The result of JavaScript's two-phase execution (creation phase + execution phase). During the creation phase, `var` declarations are registered and initialized to `undefined`, and `function` declarations are fully stored. `let`/`const` are hoisted into the **TDZ**. Code does not literally move — the environment record is populated before execution begins.

`→ See:` [`javascript-core/01-execution-context.md`](../javascript-core/01-execution-context.md)

---

## 🔒 Scope & Closures

### Lexical Scope (Static Scope)

A scoping model where a variable's scope is determined by where it appears in the source code, not where it's called at runtime. JavaScript uses lexical scope. A function's **scope chain** is fixed at definition time.

`→ See:` [`javascript-core/07-scope-chain.md`](../javascript-core/07-scope-chain.md)

---

### Scope Chain

The linked list of **lexical environments** from the current scope outward to the global scope. Variable lookup walks this chain from inner to outer, returning the first match found.

`→ See:` [`javascript-core/07-scope-chain.md`](../javascript-core/07-scope-chain.md)

---

### Closure

A function that retains access to its enclosing **lexical environment** even after the outer function has returned. The function holds a live reference (not a copy) to the environment record. Closures power private state, factory functions, memoization, and callbacks.

`→ See:` [`javascript-core/05-closures.md`](../javascript-core/05-closures.md)

---

### Closure Environment

The **lexical environment** (EnvironmentRecord + outer reference) that a closure holds a reference to. Stored on the **heap** — persists after the outer function returns as long as any closure references it. The source of closure-related **memory leaks**.

---

### Variable Shadowing

When a variable in an inner scope has the same name as one in an outer scope, the inner variable "shadows" the outer — the inner scope uses its own binding and cannot access the outer one.

---

### IIFE (Immediately Invoked Function Expression)

A function that is defined and called immediately: `(function() { ... })()`. Creates an isolated **function scope**. Was the standard pattern for scope isolation before ES modules and `let`/`const`.

---

## 🧬 Prototypes & Objects

### Prototype

An object that another object **delegates** property lookups to when the property isn't found on the object itself. Every object has an internal `[[Prototype]]` slot (accessible via `Object.getPrototypeOf()`). Forms the **prototype chain**.

`→ See:` [`javascript-core/06-prototypes.md`](../javascript-core/06-prototypes.md)

---

### Prototype Chain

The chain of `[[Prototype]]` references from an object up through its ancestors to `Object.prototype` (whose `[[Prototype]]` is `null`). Property lookups traverse this chain from innermost to outermost.

---

### `[[Prototype]]` vs `.prototype`

`[[Prototype]]` is the internal slot on every **object** that forms the prototype chain. `.prototype` is a regular property on **functions** — it becomes the `[[Prototype]]` of objects created with `new ThatFunction()`. These are different things.

`→ See:` [`javascript-core/06-prototypes.md`](../javascript-core/06-prototypes.md)

---

### Constructor Function

A regular function designed to be called with `new`. When called with `new`, the engine creates a new object with `[[Prototype]]` set to the function's `.prototype`, executes the function with `this` as the new object, and returns `this` (unless the function returns a different object).

---

### Property Shadowing

When an own property on an object has the same name as an inherited property on its prototype. The own property takes precedence (shadows) the prototype property.

---

### `hasOwnProperty` / `Object.hasOwn`

Methods to check if a property is directly on an object (not inherited via **prototype chain**). `Object.hasOwn(obj, key)` is the modern ES2022 version, safer than `obj.hasOwnProperty(key)` which can be overridden.

---

## 🗑️ Memory & Garbage Collection

### Garbage Collection (GC)

The automatic process of reclaiming memory from objects that are no longer **reachable** from any **GC root**. JavaScript uses a **mark-and-sweep** algorithm as its primary collection strategy.

`→ See:` [`javascript-core/09-garbage-collection.md`](../javascript-core/09-garbage-collection.md)

---

### GC Root

Starting points for the **garbage collector's** reachability traversal. Anything reachable from a root is kept alive. Roots include: global variables, the current call stack, active DOM nodes, event listener callbacks, pending timers, and Promise continuations.

---

### Reachability

Whether an object can be reached by traversing references from a **GC root**. Reachable = kept alive. Unreachable = eligible for collection. The GC doesn't track "is this used?" — only "can this be reached?".

---

### Memory Leak

When memory is allocated and then never freed — not because GC is broken, but because a **reference** still exists somewhere, keeping the object **reachable** even though it will never be used again.

`→ See:` [`performance/05-memory-leaks.md`](../performance/05-memory-leaks.md)

---

### Detached DOM Node

A DOM element that has been removed from the document tree but is still referenced by JavaScript. The element (and its entire subtree) cannot be **garbage collected** because the JS reference keeps it **reachable**.

---

### Mark-and-Sweep

V8's primary **garbage collection** algorithm. Mark phase: starting from **GC roots**, traverse the entire reference graph and mark all reachable objects. Sweep phase: reclaim memory from all unmarked objects. Handles **circular references** correctly (unlike reference counting).

---

### Scavenge

V8's **garbage collection** algorithm for the **young generation**. A copying collector: copies all live objects to a new semi-space (compacting them), then declares the old semi-space entirely free. Fast (~1ms) because it only processes live objects.

---

### Young Generation

The region of V8's heap where newly allocated objects are placed. Collected frequently by **Scavenge**. Objects that survive multiple collections are **promoted** to the **old generation**.

---

### Old Generation

The region of V8's heap where long-lived objects (those that survived multiple **Scavenges**) are stored. Collected by **Major GC** (Mark-Sweep-Compact). Less frequent but more expensive than young generation collection.

---

### Major GC

A full **mark-and-sweep** collection of the **old generation**. Can pause execution for 10–500ms depending on heap size. Modern V8 uses incremental marking and concurrent sweeping to minimize pause times.

---

### GC Pause

A period during which JavaScript execution is stopped while the **garbage collector** runs. Visible as dropped frames and jank. Minimized in modern V8 via incremental and concurrent collection strategies.

---

### Object Pool

A design pattern that pre-allocates objects and reuses them instead of creating new ones. Reduces **allocation rate**, reduces GC pressure, and eliminates GC pauses in animation loops.

`→ See:` [`javascript-core/09-garbage-collection.md`](../javascript-core/09-garbage-collection.md)

---

### WeakMap / WeakSet

Data structures that hold their keys **weakly** — if the key object is garbage collected, the entry is automatically removed. Unlike `Map`/`Set`, they don't prevent GC. Ideal for associating metadata with objects without causing leaks.

`→ See:` [`javascript-core/08-memory-management.md`](../javascript-core/08-memory-management.md)

---

### WeakRef

A reference to an object that does not prevent **garbage collection**. `weakRef.deref()` returns the object or `undefined` if it has been collected. Used in optional caching, component registries, and observer cleanup.

---

### Retained Size

The total memory that would be freed if a given object were garbage collected — its own size plus the size of all objects it uniquely keeps alive. The key metric for identifying high-impact memory leaks.

---

### Shallow Size

The memory directly occupied by an object itself, not including objects it references. Contrast with **retained size**.

---

## 🖥️ Browser Rendering

### Rendering Pipeline

The sequence of stages the browser executes to display a frame: Parse HTML → Build DOM → Parse CSS → Build CSSOM → Build Render Tree → Layout → Paint → Composite. Different CSS property changes trigger different stages.

`→ See:` [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md)

---

### DOM (Document Object Model)

A tree representation of an HTML document created by the browser's HTML parser. JavaScript can read and modify the DOM, which may trigger layout, paint, or composite stages.

---

### CSSOM (CSS Object Model)

A tree representation of all CSS rules with computed styles. Built from all stylesheets (external, inline, `<style>` tags). Must be fully built before the **render tree** can be constructed. CSSOM building is **render-blocking**.

---

### Render Tree

The merged result of the **DOM** and **CSSOM** trees. Contains only visible elements with their computed styles. Elements with `display: none` are excluded entirely. Elements with `visibility: hidden` or `opacity: 0` are included (they take space).

---

### Layout (Reflow)

The browser stage that computes the exact position and size (geometry) of every element. Triggered by changes that affect element dimensions or position. Expensive and cascading — one element's change can force recalculation of many others.

`→ See:` [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md)

---

### Reflow

Synonym for **Layout**. Commonly used in the context of invalidation: "triggering a reflow" means causing the browser to recompute layout.

---

### Paint (Rasterization)

The browser stage that fills in the pixels for each layer — colors, text, images, borders, shadows. Triggered by visual style changes that don't affect geometry (e.g., `color`, `background`, `box-shadow`). More expensive than **composite** but less expensive than **layout**.

---

### Repaint

Synonym for **Paint** in the context of re-doing a paint that was already done. "Triggering a repaint" means a paint stage must run again.

---

### Composite

The final browser stage. The GPU merges all painted layers into the final screen image. Only `transform` and `opacity` changes trigger composite without layout or paint. Runs on the **compositor thread**, separate from the main JavaScript thread.

`→ See:` [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md)

---

### Compositor Thread

A browser thread separate from the main JavaScript thread that handles **compositing**. Animations using only `transform` and `opacity` run entirely on this thread — they cannot be blocked by JavaScript execution or garbage collection.

---

### Layer (Compositor Layer)

A separate bitmap created for elements that need independent compositing. Elements on their own layer can be transformed or faded without triggering layout or paint. Created by `will-change`, `transform: translateZ(0)`, `position: fixed`, video/canvas elements.

---

### Layer Explosion

When too many elements are promoted to compositor **layers**, consuming excessive GPU memory and compositing overhead. Using `will-change` on many elements simultaneously is the typical cause.

---

### Critical Rendering Path (CRP)

The sequence of steps the browser must complete before rendering the first pixel: fetch HTML → parse HTML → fetch and parse CSS → build render tree → layout → paint. Optimizing the CRP reduces **First Contentful Paint (FCP)**.

`→ See:` [`browser-internals/08-critical-rendering-path.md`](../browser-internals/08-critical-rendering-path.md)

---

### Render-Blocking

A resource or operation that prevents the browser from rendering any content until it completes. CSS files are render-blocking by default. JavaScript `<script>` tags are parser-blocking AND render-blocking unless `async` or `defer`.

---

### Layout Thrashing (Forced Synchronous Layout)

A performance anti-pattern where JavaScript alternates reading layout-dependent DOM properties (like `offsetWidth`) with writing style changes. Each read forces an immediate layout recalculation, potentially causing dozens of full layout passes in a single frame.

`→ See:` [`performance/03-layout-thrashing.md`](../performance/03-layout-thrashing.md)

---

### 16ms Frame Budget

At 60fps, the browser has 16.67ms to complete a full frame (JavaScript + style + layout + paint + composite). Any operation that exceeds this budget causes dropped frames and visible **jank**.

---

### Jank

Visible stuttering or hitching in animations or scrolling, caused by frames taking longer than ~16ms. Typically caused by **long tasks**, **layout thrashing**, or large **GC pauses**.

---

### CSS Containment

A CSS feature (`contain: layout | paint | size | strict | content`) that tells the browser changes inside an element cannot affect elements outside it. Limits the scope of **reflow** and repaint, improving performance.

---

### will-change

A CSS property that hints to the browser that an element will be animated. The browser can promote the element to its own compositor **layer** ahead of time, avoiding the cost of promotion during animation. Must be used sparingly — creates memory overhead.

---

## ⚡ Performance

### RAIL Model

Google's performance framework: **Response** (< 100ms for user input), **Animation** (frames in 16ms), **Idle** (use idle time for proactive work), **Load** (interactive in < 5 seconds). Provides targets for each type of user interaction.

---

### Core Web Vitals (CWV)

Google's set of user-centric performance metrics:

- **LCP (Largest Contentful Paint):** Time until largest visible content renders. Target: < 2.5s
- **FID (First Input Delay) / INP (Interaction to Next Paint):** Input responsiveness. Target: < 100ms / < 200ms
- **CLS (Cumulative Layout Shift):** Visual stability — unexpected layout shifts. Target: < 0.1

---

### FPS (Frames Per Second)

The number of frames the browser renders per second. 60fps = 16.67ms per frame. Smooth animation requires consistent 60fps. Drops below 60fps (especially erratic drops) are perceived as **jank**.

---

### Virtualization (Windowing)

A technique for rendering large lists or grids where only the visible items are in the DOM. As the user scrolls, items outside the viewport are unmounted and replaced by newly visible items. Reduces DOM size from thousands to tens of nodes.

`→ See:` [`performance/02-virtualization-windowing.md`](../performance/02-virtualization-windowing.md)

---

### Memoization

Caching the results of function calls: if the same inputs are provided again, return the cached result instead of recomputing. Reduces CPU work at the cost of memory. Correct when: function is pure (same inputs always produce same output) and calls are expensive.

`→ See:` [`performance/07-memoization.md`](../performance/07-memoization.md)

---

### Event Delegation

Attaching a single event listener to a parent element instead of individual listeners to each child. Uses event **bubbling** to catch events from descendants. More efficient for dynamic lists — one listener vs N listeners.

`→ See:` [`performance/06-event-delegation.md`](../performance/06-event-delegation.md)

---

### Debounce

A technique that delays execution of a function until a specified time has passed since the last call. Useful for high-frequency events (search input, window resize) where only the final value matters.

`→ See:` [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md)

---

### Throttle

A technique that limits execution of a function to at most once per time interval. Unlike **debounce**, throttle guarantees periodic execution — it doesn't reset on each call.

---

### Tree Shaking

A build optimization that removes unused exports from JavaScript bundles. Works with static ES module `import`/`export` syntax — the bundler can statically determine what's used and exclude the rest.

---

### Code Splitting

Breaking a JavaScript bundle into smaller chunks loaded on demand. Reduces initial load time — only the code needed for the current page is loaded. Implemented via dynamic `import()`.

---

### Lazy Loading

Deferring the loading of a resource (image, script, component, data) until it's actually needed — typically when it enters the viewport. Reduces initial page load time.

---

### Prefetching

Loading a resource before it's needed, during idle time. Improves subsequent navigation speed by having resources ready in the browser cache.

---

### LRU Cache (Least Recently Used)

A cache eviction strategy that removes the least recently accessed entry when the cache is full. Maintains a fixed maximum size. Commonly implemented with a `Map` (which preserves insertion order) for O(1) get/set.

---

## 🏗️ Architecture

### Micro-Frontend

An architectural pattern where a frontend application is split into independently deployable pieces, each owned by a separate team. Each piece can use different frameworks, have independent deployments, and communicate via a shared event bus or module federation.

`→ See:` [`system-design/03-micro-frontends.md`](../system-design/03-micro-frontends.md)

---

### Module Federation

A Webpack/Vite feature that allows JavaScript modules to be loaded at runtime from a remote URL. Enables **micro-frontends** to share code without bundling it into each app's bundle.

---

### Feature-Based Architecture

An application structure organized around product features (e.g., `/features/auth/`, `/features/cart/`) rather than technical roles (e.g., `/components/`, `/services/`). Improves code discoverability and team ownership at scale.

`→ See:` [`system-design/02-feature-based-structure.md`](../system-design/02-feature-based-structure.md)

---

### Config-Driven UI

A UI architecture where the layout, behavior, and content of a screen are determined by a data structure (the "config") rather than hardcoded component trees. Enables dynamic form builders, CMS-driven pages, and A/B testing at the configuration level.

`→ See:` [`system-design/05-config-driven-ui.md`](../system-design/05-config-driven-ui.md)

---

### Plugin Architecture

A system where functionality can be extended at runtime by registering "plugins" — independent modules that hook into predefined extension points. Enables extensibility without modifying the core system.

`→ See:` [`architecture/04-plugin-architecture.md`](../architecture/04-plugin-architecture.md)

---

### Reactive System

A system that automatically propagates state changes to all interested dependents. When state changes, the UI (or any subscriber) updates automatically without manual notification. Vue's reactivity system, MobX, and RxJS are examples.

`→ See:` [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md)

---

### State Management

The discipline of organizing, storing, and updating application state in a predictable, maintainable way. Includes decisions about where state lives (local vs global), how it flows (unidirectional vs bidirectional), and how updates are triggered.

`→ See:` [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md)

---

### Unidirectional Data Flow

An architecture pattern where data flows in one direction: state → view → actions → state. Mutations only happen through explicit action dispatches. Redux and Flux follow this pattern. Makes state changes predictable and debuggable.

---

### Event-Driven Architecture

A software design pattern where components communicate by emitting and listening to events rather than calling each other directly. Decouples producers from consumers. The **Event Bus** / **Pub/Sub** pattern is the primary mechanism.

`→ See:` [`system-design/06-event-driven-frontend.md`](../system-design/06-event-driven-frontend.md)

---

## 🌐 Browser APIs

### Web Worker

A JavaScript thread that runs in parallel with the main thread. Has its own heap, call stack, and event loop. Cannot access the DOM. Communicates with the main thread only via `postMessage`. Ideal for CPU-intensive work that would otherwise block the UI.

`→ See:` [`javascript-core/12-web-workers.md`](../javascript-core/12-web-workers.md)

---

### Shared Worker

A Web Worker that can be accessed by multiple pages, tabs, or iframes from the same origin. Useful for sharing a WebSocket connection or synchronized state across browser tabs.

---

### Service Worker

A scriptable network proxy that intercepts fetch requests, enables offline caching, and can receive push notifications even when the page is closed. Requires HTTPS. Core technology for Progressive Web Apps (PWA).

`→ See:` [`javascript-core/13-service-workers.md`](../javascript-core/13-service-workers.md)

---

### MutationObserver

A browser API that asynchronously notifies of changes to the DOM tree (node additions/removals, attribute changes, text changes). More efficient than polling. Callbacks are **microtasks**.

`→ See:` [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md)

---

### IntersectionObserver

A browser API that asynchronously observes changes in the intersection of an element with the viewport or a parent container. Used for lazy loading, infinite scroll, and sticky header detection. Replaces scroll-event-based visibility checks.

`→ See:` [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md)

---

### ResizeObserver

A browser API that fires when an element's size changes. More efficient than window `resize` events for component-level responsiveness. Fires at the end of the rendering pipeline.

`→ See:` [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md)

---

### AbortController

A Web API for cancelling asynchronous operations. `controller.abort()` sends an abort signal to any operation listening to `controller.signal` — including `fetch` requests and `addEventListener` calls (via `{ signal }` option).

`→ See:` [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md)

---

### Structured Clone Algorithm

The algorithm used by `postMessage` to deep-copy data sent between threads or contexts. Handles more types than JSON (Date, Map, Set, TypedArray, circular refs) but cannot copy functions or DOM nodes.

`→ See:` [`javascript-core/12-web-workers.md`](../javascript-core/12-web-workers.md)

---

### Transferable Object

An object that can be _moved_ (not copied) between threads via `postMessage`. After transfer, the original reference is detached (unusable). Includes `ArrayBuffer`, `MessagePort`, `ImageBitmap`. Provides zero-copy data transfer.

---

### SharedArrayBuffer

A fixed-length binary data buffer shared across threads. Unlike regular `ArrayBuffer`, it is not copied on transfer — both threads access the same memory. Requires `Atomics` for thread-safe reads and writes.

---

### Cache API

A browser API for storing **Request → Response** pairs. Available in Service Workers and regular pages. Unlike HTTP cache (controlled by headers), the Cache API is fully programmable — you decide what to cache, how long to keep it, and when to invalidate.

`→ See:` [`javascript-core/13-service-workers.md`](../javascript-core/13-service-workers.md)

---

### IndexedDB

A browser-native NoSQL database for storing large amounts of structured data client-side. Supports indexes, transactions, and asynchronous operations. Available in Workers. Used for offline data, caching, and persistent application state.

---

### OffscreenCanvas

A canvas element that can be used in a **Web Worker**, enabling complex 2D or WebGL rendering off the main thread. The `transferControlToOffscreen()` method transfers rendering control from the main thread to a worker.

---

## 🔀 Promises & Async

### Promise

An object representing a value that will be available in the future. Has three states: **pending** (initial), **fulfilled** (resolved with a value), or **rejected** (failed with a reason). State transitions are one-way and permanent.

`→ See:` [`javascript-core/11-promise-internals.md`](../javascript-core/11-promise-internals.md)

---

### Thenable

Any object or function with a `.then` method. The Promise resolution procedure treats thenables like native Promises — the chain waits for them to settle. This is how Promise interoperability works across libraries.

---

### PromiseResolveThenableJob

An internal V8/spec job created when a `.then()` handler returns a thenable. This job calls the thenable's `.then()` method, adding two extra **microtask** ticks to the chain compared to returning a plain value.

`→ See:` [`javascript-core/11-promise-internals.md`](../javascript-core/11-promise-internals.md)

---

### Async/Await

Syntax sugar over Promises. An `async` function always returns a Promise. `await` suspends the async function and schedules the continuation as a **microtask** when the awaited Promise settles. Does not block the main thread.

`→ See:` [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md)

---

### Unhandled Rejection

A rejected Promise with no `.catch()` or rejection handler attached before the browser's tracking deadline (end of the current **microtask** checkpoint). Triggers the `unhandledrejection` event on `window`.

---

### Backpressure

The problem of a producer generating work faster than a consumer can process it. In async systems, uncontrolled parallel operations (e.g., launching 1000 fetch requests simultaneously) can overwhelm resources. Solved by concurrency limiting.

---

## 🏛️ Design Patterns

### Observer Pattern

A behavioral design pattern where a **subject** maintains a list of **observers** and notifies them directly when its state changes. Subject holds direct references to observers. Compare with **Pub/Sub**.

`→ See:` [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md)

---

### Pub/Sub (Publish/Subscribe)

A messaging pattern with a central **event bus** between publishers and subscribers. Publishers emit events to the bus; subscribers register for channels on the bus. Neither knows about the other — fully decoupled.

`→ See:` [`javascript-core/15-pub-sub-systems.md`](../javascript-core/15-pub-sub-systems.md)

---

### Module Pattern

Using a **closure** (often an **IIFE**) to create a private scope with a public API. Private variables are inaccessible from outside. The foundation of JavaScript's encapsulation before ES modules and private class fields.

---

### Proxy Pattern

Using JavaScript's `Proxy` object to intercept operations on another object (property reads, writes, function calls). Powers Vue 3's reactivity system, schema validation, and access logging.

`→ See:` [`patterns/05-proxy-pattern.md`](../patterns/05-proxy-pattern.md)

---

### Factory Pattern

A function or class that creates and returns objects without exposing the creation logic. Allows choosing the type of object to create based on input, and hides the constructor from callers.

---

### Singleton Pattern

Ensuring a class or object has only one instance, with a global access point. Common for event buses, configuration stores, and connection pools.

---

### Command Pattern

Encapsulating a request as an object, allowing it to be queued, logged, undone, or replayed. Useful for undo/redo systems, task queues, and macro recording.

`→ See:` [`patterns/03-command.md`](../patterns/03-command.md)

---

### Strategy Pattern

Defining a family of algorithms, encapsulating each one, and making them interchangeable. Allows the algorithm to vary independently from the clients that use it. Example: caching strategies in Service Workers.

`→ See:` [`patterns/04-strategy.md`](../patterns/04-strategy.md)

---

## 📡 Networking

### HTTP Caching

Browser-native caching controlled by HTTP response headers (`Cache-Control`, `ETag`, `Last-Modified`). Distinct from the **Cache API** — HTTP cache is automatic and follows header rules; Cache API is fully programmable.

`→ See:` [`caching/01-http-caching.md`](../caching/01-http-caching.md)

---

### Cache-Control

An HTTP response header that controls caching behavior. Key directives: `max-age` (seconds before stale), `no-cache` (revalidate before using cached), `no-store` (never cache), `immutable` (content never changes — skip revalidation for max-age duration).

---

### ETag

An HTTP response header containing a version identifier for a resource. The browser sends `If-None-Match: <etag>` on subsequent requests. If the resource hasn't changed, the server returns `304 Not Modified` with no body — saving bandwidth.

---

### WebSocket

A protocol providing full-duplex (bidirectional) communication over a single TCP connection. Unlike HTTP (request/response), either side can send messages at any time. Used for real-time features: live dashboards, chat, collaborative editing.

`→ See:` [`networking/02-websockets.md`](../networking/02-websockets.md)

---

### CORS (Cross-Origin Resource Sharing)

A browser security mechanism that restricts cross-origin HTTP requests. Requests from one origin (domain + port + protocol) to another require the server to include `Access-Control-Allow-Origin` headers. Preflight OPTIONS requests are sent for non-simple requests.

`→ See:` [`security/03-cors.md`](../security/03-cors.md)

---

### CSP (Content Security Policy)

An HTTP response header that restricts the sources from which scripts, styles, images, and other resources can be loaded. Primary defense against **XSS** attacks. Specified via `Content-Security-Policy` header or `<meta>` tag.

`→ See:` [`security/02-csp.md`](../security/02-csp.md)

---

### XSS (Cross-Site Scripting)

A security vulnerability where attackers inject malicious scripts into web pages viewed by other users. Prevented by: output encoding, **CSP**, avoiding `innerHTML` with untrusted data, and sanitizing user input.

`→ See:` [`security/01-xss.md`](../security/01-xss.md)

---

## 📐 Rendering Architectures

### SSR (Server-Side Rendering)

Rendering HTML on the server for each request. The browser receives fully formed HTML, enabling fast **First Contentful Paint**. JavaScript then **hydrates** the page for interactivity.

`→ See:` [`browser-internals/10-ssr-csr-isr-streaming.md`](../browser-internals/10-ssr-csr-isr-streaming.md)

---

### CSR (Client-Side Rendering)

Rendering the entire application in the browser using JavaScript. The server sends a minimal HTML shell; JavaScript fetches data and builds the DOM. Slower first load but fast subsequent navigation.

---

### ISR (Incremental Static Regeneration)

A Next.js rendering strategy that pre-renders pages at build time but allows individual pages to be regenerated (revalidated) at runtime at a specified interval, without a full rebuild.

---

### Hydration

The process of attaching JavaScript event listeners and state to **SSR**-generated HTML. The browser has the visible HTML immediately (from SSR) and then "hydrates" it to make it interactive.

---

### Streaming SSR

Server-side rendering where the server streams HTML chunks to the browser as they're ready, rather than waiting for the full page. The browser can start rendering and display visible content before the complete HTML arrives.

---

### PWA (Progressive Web App)

A web application that uses modern browser capabilities (**Service Workers**, **Web App Manifest**, HTTPS) to provide app-like experiences: offline support, installability, push notifications, and fast loading.

---

## 🧮 Canvas & Visualization

### Canvas API

A browser API for drawing 2D graphics via JavaScript. Low-level but highly performant for dynamic visualizations, games, and image processing. Everything drawn is rasterized — there are no DOM elements to query after drawing.

`→ See:` [`performance/10-canvas-optimization.md`](../performance/10-canvas-optimization.md)

---

### Dirty Rectangle

An optimization technique for canvas rendering where only the areas of the canvas that have changed ("dirty rects") are redrawn, rather than clearing and redrawing the entire canvas each frame.

---

### WebGL

A browser API for rendering 2D and 3D graphics using the GPU directly via OpenGL ES. Far more performant than Canvas 2D for complex scenes. Used by Three.js, Babylon.js, and high-performance data visualizations.

---

### SVG (Scalable Vector Graphics)

An XML-based format for 2D vector graphics. SVG elements are DOM nodes — they can be styled with CSS, queried with JavaScript, and animated. Good for hundreds of elements; degrades for thousands.

`→ See:` [`performance/11-svg-optimization.md`](../performance/11-svg-optimization.md)

---

_This glossary is a living document. If you encounter a term used in this handbook that isn't defined here, please [open an issue](../../issues) or submit a PR._
