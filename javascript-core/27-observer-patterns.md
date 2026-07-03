# 14 — Observer Patterns

> **"The Observer pattern is how loosely coupled systems talk to each other. Instead of A calling B directly, A says 'something happened' and B — if it cares — responds. That decoupling is the foundation of reactive systems, event-driven architecture, and modern UI frameworks."**

Observer patterns appear everywhere in frontend development: DOM events, React's state model, Redux, RxJS, Vue's reactivity system, MobX, EventEmitter, WebSocket handlers, IntersectionObserver, ResizeObserver, MutationObserver. This document covers the full spectrum — from the basic pattern to production-grade implementations, reactive systems, and the browser's native observer APIs.

---

## 📚 Table of Contents

1. [The Observer Pattern — Core Concept](#1-the-observer-pattern--core-concept)
2. [Classic Implementation](#2-classic-implementation)
3. [EventEmitter — Node.js Style](#3-eventemitter--nodejs-style)
4. [Typed Event System](#4-typed-event-system)
5. [Reactive State with Observers](#5-reactive-state-with-observers)
6. [The Proxy-Based Reactive System](#6-the-proxy-based-reactive-system)
7. [Browser Native Observers](#7-browser-native-observers)
8. [MutationObserver](#8-mutationobserver)
9. [IntersectionObserver](#9-intersectionobserver)
10. [ResizeObserver](#10-resizeobserver)
11. [PerformanceObserver](#11-performanceobserver)
12. [Observable Pattern (RxJS-style)](#12-observable-pattern-rxjs-style)
13. [Observer vs Pub/Sub — The Difference](#13-observer-vs-pubsub--the-difference)
14. [Memory Management in Observer Systems](#14-memory-management-in-observer-systems)
15. [Good Practices](#15-good-practices)
16. [Bad Practices](#16-bad-practices)
17. [Common Mistakes](#17-common-mistakes)
18. [Interview-Level Explanation](#18-interview-level-explanation)
19. [Exercises](#19-exercises)

---

## 1. The Observer Pattern — Core Concept

The Observer pattern defines a one-to-many dependency: when one object (the **Subject** or **Observable**) changes state, all dependent objects (**Observers**) are automatically notified and updated.

```
WITHOUT Observer (tight coupling):
  Counter.increment() → directly calls button.update()
                     → directly calls display.render()
                     → directly calls logger.log()
  Counter knows about all its dependents — fragile, hard to extend

WITH Observer (loose coupling):
  Counter.increment() → notifies('change', newValue)
                     ← button subscribed to 'change' → updates itself
                     ← display subscribed to 'change' → renders itself
                     ← logger subscribed to 'change' → logs itself
  Counter knows nothing about its observers — extensible, decoupled
```

### Key Participants

```
Subject (Observable):
  - Maintains a list of observers
  - Provides subscribe/unsubscribe methods
  - Notifies observers when state changes

Observer:
  - Registers with a subject
  - Has an update() method called when subject changes
  - Can unsubscribe when no longer interested

Notification:
  - Data passed from subject to observer
  - Can be the full state, a diff, or an event object
```

---

## 2. Classic Implementation

```javascript
class Subject {
  #observers = new Set();
  #state;

  constructor(initialState) {
    this.#state = initialState;
  }

  /**
   * Register an observer function.
   * Returns an unsubscribe function.
   */
  subscribe(observer) {
    if (typeof observer !== "function") {
      throw new TypeError("Observer must be a function");
    }
    this.#observers.add(observer);

    // Return unsubscribe function — crucial for memory management
    return () => this.#observers.delete(observer);
  }

  /**
   * Notify all observers with the current state.
   */
  #notify(data) {
    // Snapshot observers before iterating — safe if observers unsubscribe during notification
    for (const observer of [...this.#observers]) {
      try {
        observer(data);
      } catch (err) {
        // Don't let one observer crash others
        console.error("Observer threw an error:", err);
      }
    }
  }

  /**
   * Update state and notify all observers.
   */
  setState(newState) {
    const prevState = this.#state;
    this.#state = newState;

    // Only notify if state actually changed (shallow comparison)
    if (prevState !== newState) {
      this.#notify({ prev: prevState, next: newState });
    }
  }

  getState() {
    return this.#state;
  }

  get observerCount() {
    return this.#observers.size;
  }
}

// Usage
const counter = new Subject(0);

const unsubA = counter.subscribe(({ next }) => {
  console.log("Observer A:", next);
});

const unsubB = counter.subscribe(({ next }) => {
  document.getElementById("counter").textContent = next;
});

counter.setState(1); // notifies A and B
counter.setState(2); // notifies A and B
counter.setState(2); // no notification — state didn't change

unsubA(); // A unsubscribes
counter.setState(3); // only B notified
```

---

## 3. EventEmitter — Node.js Style

EventEmitter is a named-event variant of the Observer pattern. Instead of a single notification, events are categorized by name.

```javascript
class EventEmitter {
  #events = new Map(); // eventName → Set<handler>
  #maxListeners = 10; // warn on potential leak

  /**
   * Register a handler for an event.
   * Returns an unsubscribe function.
   */
  on(event, handler) {
    if (!this.#events.has(event)) {
      this.#events.set(event, new Set());
    }

    const handlers = this.#events.get(event);

    if (handlers.size >= this.#maxListeners) {
      console.warn(
        `EventEmitter: possible memory leak. ${handlers.size} listeners for "${event}"`,
      );
    }

    handlers.add(handler);

    return () => this.off(event, handler);
  }

  /**
   * Register a one-time handler — auto-removes after first fire.
   */
  once(event, handler) {
    const wrapper = (...args) => {
      handler(...args);
      this.off(event, wrapper);
    };
    wrapper._original = handler; // for removeListener by original fn
    return this.on(event, wrapper);
  }

  /**
   * Remove a specific handler.
   */
  off(event, handler) {
    const handlers = this.#events.get(event);
    if (!handlers) return this;

    // Handle both direct handlers and once-wrappers
    for (const h of handlers) {
      if (h === handler || h._original === handler) {
        handlers.delete(h);
        break;
      }
    }

    if (handlers.size === 0) {
      this.#events.delete(event);
    }

    return this;
  }

  /**
   * Emit an event, calling all registered handlers.
   */
  emit(event, ...args) {
    const handlers = this.#events.get(event);
    if (!handlers || handlers.size === 0) return false;

    // Snapshot for safe iteration (handlers may unsubscribe during emit)
    for (const handler of [...handlers]) {
      try {
        handler(...args);
      } catch (err) {
        this.emit("error", err); // route errors to error handlers
      }
    }

    return true;
  }

  /**
   * Remove all handlers for an event, or all events.
   */
  removeAllListeners(event) {
    if (event) {
      this.#events.delete(event);
    } else {
      this.#events.clear();
    }
    return this;
  }

  /**
   * Get all registered event names.
   */
  eventNames() {
    return [...this.#events.keys()];
  }

  listenerCount(event) {
    return this.#events.get(event)?.size ?? 0;
  }

  setMaxListeners(n) {
    this.#maxListeners = n;
    return this;
  }
}

// Usage
const emitter = new EventEmitter();

const unsubscribe = emitter.on("data", (payload) => {
  console.log("Received:", payload);
});

emitter.once("connect", () => {
  console.log("Connected! (fires once)");
});

emitter.emit("connect"); // 'Connected!'
emitter.emit("connect"); // silence — once handler removed itself
emitter.emit("data", { id: 1 }); // 'Received: { id: 1 }'

unsubscribe(); // clean up
```

---

## 4. Typed Event System

For large TypeScript/JavaScript codebases, a typed event system prevents bugs by enforcing correct event names and payload types.

```javascript
/**
 * Strongly typed event emitter.
 * Define event types as a record of event name → payload type.
 *
 * Usage (TypeScript):
 *   type AppEvents = {
 *     'user:login': { userId: string; timestamp: number };
 *     'cart:update': { items: CartItem[]; total: number };
 *     'error': Error;
 *   };
 *   const events = new TypedEventEmitter<AppEvents>();
 *   events.on('user:login', ({ userId }) => ...);       // ✅ typed
 *   events.on('user:lgoin', handler);                   // ❌ TS error
 *   events.emit('user:login', { userId: '42', timestamp: Date.now() }); // ✅
 *   events.emit('user:login', { userId: 42 });          // ❌ TS error
 */
class TypedEventEmitter {
  #handlers = new Map();

  on(event, handler) {
    if (!this.#handlers.has(event)) this.#handlers.set(event, new Set());
    this.#handlers.get(event).add(handler);
    return () => this.#handlers.get(event)?.delete(handler);
  }

  once(event, handler) {
    const unsub = this.on(event, function onceHandler(...args) {
      handler(...args);
      unsub();
    });
    return unsub;
  }

  emit(event, payload) {
    this.#handlers.get(event)?.forEach((h) => {
      try {
        h(payload);
      } catch (err) {
        console.error(`Error in handler for "${event}":`, err);
      }
    });
  }

  off(event, handler) {
    this.#handlers.get(event)?.delete(handler);
  }

  clear() {
    this.#handlers.clear();
  }
}

// Domain event bus example
const appEvents = new TypedEventEmitter();

// Subscribe
const unsub = appEvents.on("user:login", ({ userId, timestamp }) => {
  analytics.track("login", { userId });
  loadUserPreferences(userId);
});

// Emit
appEvents.emit("user:login", { userId: "42", timestamp: Date.now() });

// Cleanup
unsub();
```

---

## 5. Reactive State with Observers

A reactive store combines state management with the observer pattern — observers are automatically notified when relevant state changes.

```javascript
class Store {
  #state;
  #listeners = new Map(); // path → Set<listener>
  #globalListeners = new Set();
  #batching = false;
  #pendingPaths = new Set();

  constructor(initialState) {
    this.#state = structuredClone(initialState);
  }

  /**
   * Get a value from the store by path (e.g., 'user.name')
   */
  get(path) {
    return path.split(".").reduce((obj, key) => obj?.[key], this.#state);
  }

  /**
   * Set a value and notify relevant observers.
   */
  set(path, value) {
    const keys = path.split(".");
    const last = keys.pop();
    const target = keys.reduce((obj, key) => {
      if (obj[key] === undefined || typeof obj[key] !== "object") {
        obj[key] = {};
      }
      return obj[key];
    }, this.#state);

    const prev = target[last];
    if (prev === value) return; // no change

    target[last] = value;

    if (this.#batching) {
      this.#pendingPaths.add(path);
    } else {
      this.#notifyPath(path);
      this.#notifyGlobal();
    }
  }

  /**
   * Batch multiple updates — only notify once at the end.
   */
  batch(fn) {
    this.#batching = true;
    try {
      fn();
    } finally {
      this.#batching = false;
      // Notify all changed paths
      this.#pendingPaths.forEach((path) => this.#notifyPath(path));
      this.#pendingPaths.clear();
      if (this.#pendingPaths.size > 0) this.#notifyGlobal();
    }
  }

  /**
   * Subscribe to changes at a specific path.
   */
  watch(path, listener) {
    if (!this.#listeners.has(path)) this.#listeners.set(path, new Set());
    this.#listeners.get(path).add(listener);
    return () => this.#listeners.get(path)?.delete(listener);
  }

  /**
   * Subscribe to ALL state changes.
   */
  subscribe(listener) {
    this.#globalListeners.add(listener);
    return () => this.#globalListeners.delete(listener);
  }

  #notifyPath(path) {
    const listeners = this.#listeners.get(path);
    if (!listeners) return;
    const value = this.get(path);
    for (const listener of [...listeners]) {
      try {
        listener(value, path);
      } catch (err) {
        console.error("Store listener error:", err);
      }
    }
  }

  #notifyGlobal() {
    const snapshot = structuredClone(this.#state);
    for (const listener of [...this.#globalListeners]) {
      try {
        listener(snapshot);
      } catch (err) {
        console.error("Store global listener error:", err);
      }
    }
  }

  getSnapshot() {
    return structuredClone(this.#state);
  }
}

// Usage
const store = new Store({
  user: { name: "Alice", role: "admin" },
  cart: { items: [], total: 0 },
  theme: "dark",
});

// Watch a specific path
const unwatch = store.watch("user.name", (name) => {
  document.getElementById("username").textContent = name;
});

// Batch multiple updates (one notification)
store.batch(() => {
  store.set("user.name", "Bob");
  store.set("theme", "light");
  store.set("cart.total", 99.99);
});
```

---

## 6. The Proxy-Based Reactive System

Vue 3's reactivity system uses JavaScript `Proxy` to intercept property access and mutation, enabling fine-grained reactivity without explicit subscriptions.

```javascript
class ReactiveSystem {
  #targetMap = new WeakMap(); // target → Map<key, Set<effect>>
  #activeEffect = null; // currently running effect

  /**
   * Create a reactive proxy of an object.
   * Property reads are tracked. Property writes trigger effects.
   */
  reactive(target) {
    return new Proxy(target, {
      get: (obj, key) => {
        this.#track(obj, key); // track which effect is reading this
        const value = Reflect.get(obj, key);

        // Deep reactivity — wrap nested objects too
        if (value !== null && typeof value === "object") {
          return this.reactive(value);
        }
        return value;
      },

      set: (obj, key, value) => {
        const result = Reflect.set(obj, key, value);
        this.#trigger(obj, key); // notify effects that read this key
        return result;
      },

      deleteProperty: (obj, key) => {
        const result = Reflect.deleteProperty(obj, key);
        this.#trigger(obj, key);
        return result;
      },
    });
  }

  /**
   * Run an effect function. Re-runs automatically when reactive
   * properties it accesses are changed.
   */
  effect(fn) {
    const effectFn = () => {
      this.#activeEffect = effectFn;
      try {
        fn(); // run — any reactive reads inside are tracked
      } finally {
        this.#activeEffect = null;
      }
    };

    effectFn(); // run immediately to collect dependencies
    return effectFn;
  }

  /**
   * Like effect, but only re-runs when dependencies change (lazy).
   */
  computed(fn) {
    let cached;
    let dirty = true;

    const update = this.effect(() => {
      dirty = true;
    });

    return {
      get value() {
        if (dirty) {
          cached = fn();
          dirty = false;
        }
        return cached;
      },
    };
  }

  #track(target, key) {
    if (!this.#activeEffect) return; // not inside an effect — skip

    if (!this.#targetMap.has(target)) {
      this.#targetMap.set(target, new Map());
    }

    const depsMap = this.#targetMap.get(target);

    if (!depsMap.has(key)) {
      depsMap.set(key, new Set());
    }

    depsMap.get(key).add(this.#activeEffect);
  }

  #trigger(target, key) {
    const depsMap = this.#targetMap.get(target);
    if (!depsMap) return;

    const effects = depsMap.get(key);
    if (!effects) return;

    // Run all effects that depend on this key
    for (const effect of [...effects]) {
      effect();
    }
  }
}

// Usage
const rs = new ReactiveSystem();
const state = rs.reactive({ count: 0, name: "Alice" });

// Effect: runs immediately + re-runs whenever count changes
rs.effect(() => {
  console.log("Count is:", state.count);
});

// Computed: lazy, only recalculates when dependencies change
const doubled = rs.computed(() => state.count * 2);

state.count = 1; // logs: 'Count is: 1'
state.count = 2; // logs: 'Count is: 2'
console.log(doubled.value); // 4

state.name = "Bob"; // does NOT re-run the count effect
```

---

## 7. Browser Native Observers

The browser provides several built-in observer APIs that follow the observer pattern. They are more efficient than polling or event listeners for their specific use cases.

```
┌─────────────────────────────────────────────────────────────────┐
│                   BROWSER NATIVE OBSERVERS                       │
│                                                                   │
│  MutationObserver     — DOM tree changes                        │
│  IntersectionObserver — Element visibility (viewport/container) │
│  ResizeObserver       — Element size changes                    │
│  PerformanceObserver  — Performance metrics (long tasks, etc.)  │
│  ReportingObserver    — Browser deprecation/intervention reports│
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. MutationObserver

Watches for changes to the DOM tree — node additions/removals, attribute changes, text content changes.

```javascript
// Basic usage
const observer = new MutationObserver((mutations) => {
  for (const mutation of mutations) {
    console.log("Mutation type:", mutation.type);

    if (mutation.type === "childList") {
      console.log("Added nodes:", mutation.addedNodes);
      console.log("Removed nodes:", mutation.removedNodes);
    }

    if (mutation.type === "attributes") {
      console.log("Attribute changed:", mutation.attributeName);
      console.log("Old value:", mutation.oldValue);
    }

    if (mutation.type === "characterData") {
      console.log("Text changed. Old:", mutation.oldValue);
    }
  }
});

// Configure what to observe
observer.observe(targetElement, {
  childList: true, // observe child node additions/removals
  subtree: true, // observe all descendants (not just direct children)
  attributes: true, // observe attribute changes
  attributeOldValue: true, // include old attribute value
  characterData: true, // observe text content changes
  characterDataOldValue: true,
  attributeFilter: ["class", "data-state"], // only these attributes
});

// Stop observing
observer.disconnect();
```

### Real-World Use Cases

```javascript
// Use case 1: Detect third-party DOM injections
const observer = new MutationObserver((mutations) => {
  for (const mutation of mutations) {
    for (const node of mutation.addedNodes) {
      if (node.nodeType === Node.ELEMENT_NODE) {
        if (node.matches(".ad-injection, [data-ad]")) {
          console.warn("Ad injected:", node);
          node.remove(); // block unwanted injections
        }
      }
    }
  }
});

observer.observe(document.body, { childList: true, subtree: true });
```

```javascript
// Use case 2: Watch for dynamic content to initialize components
const observer = new MutationObserver((mutations) => {
  for (const mutation of mutations) {
    for (const node of mutation.addedNodes) {
      if (node.nodeType !== Node.ELEMENT_NODE) continue;

      // Initialize datepicker on any new date inputs
      node.querySelectorAll?.("[data-datepicker]").forEach((el) => {
        if (!el._datepickerInit) {
          initDatepicker(el);
          el._datepickerInit = true;
        }
      });
    }
  }
});

observer.observe(document.body, { childList: true, subtree: true });
```

```javascript
// Use case 3: Attribute-based state tracking
const observer = new MutationObserver((mutations) => {
  for (const mutation of mutations) {
    if (
      mutation.type === "attributes" &&
      mutation.attributeName === "data-state"
    ) {
      const newState = mutation.target.getAttribute("data-state");
      handleStateChange(mutation.target, newState, mutation.oldValue);
    }
  }
});

observer.observe(dialog, {
  attributes: true,
  attributeFilter: ["data-state"],
  attributeOldValue: true,
});
```

### MutationObserver Performance

```javascript
// ❌ Observing too broadly causes many unnecessary callbacks
observer.observe(document.body, {
  childList: true,
  subtree: true, // entire document tree
  attributes: true, // all attributes
  characterData: true, // all text changes
});
// Any keystroke in any text input fires this callback

// ✅ Observe only what you need, as narrowly as possible
observer.observe(specificContainer, {
  childList: true, // only node additions/removals
  subtree: false, // only direct children
  // no attributes, no characterData
});
```

---

## 9. IntersectionObserver

Asynchronously observes changes in the intersection of a target element with an ancestor element or the viewport. Replaces scroll-based visibility detection.

```javascript
// Basic usage
const observer = new IntersectionObserver(
  (entries) => {
    for (const entry of entries) {
      // entry.isIntersecting: true if any part is visible
      // entry.intersectionRatio: 0.0–1.0 (fraction visible)
      // entry.boundingClientRect: target's bounding rect
      // entry.intersectionRect: visible portion
      // entry.rootBounds: viewport rect
      // entry.target: the observed element
      // entry.time: when the intersection changed (DOMHighResTimeStamp)

      if (entry.isIntersecting) {
        console.log("Element entered viewport:", entry.target);
      } else {
        console.log("Element left viewport:", entry.target);
      }
    }
  },
  {
    root: null, // null = viewport
    rootMargin: "0px", // expand/contract root bounds (like CSS margin)
    threshold: 0.5, // fire when 50% visible (0.0–1.0, or array)
    // threshold: [0, 0.25, 0.5, 0.75, 1.0] — fire at each threshold
  },
);

observer.observe(element);
observer.unobserve(element); // stop observing specific element
observer.disconnect(); // stop observing all elements
```

### Lazy Image Loading

```javascript
function lazyLoadImages(selector = "img[data-src]") {
  const observer = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        if (!entry.isIntersecting) continue;

        const img = entry.target;
        const src = img.dataset.src;
        const srcset = img.dataset.srcset;

        if (src) img.src = src;
        if (srcset) img.srcset = srcset;

        img.removeAttribute("data-src");
        img.removeAttribute("data-srcset");

        observer.unobserve(img); // unobserve once loaded — memory efficient
      }
    },
    {
      rootMargin: "200px 0px", // start loading 200px before entering viewport
      threshold: 0,
    },
  );

  document.querySelectorAll(selector).forEach((img) => observer.observe(img));

  return () => observer.disconnect(); // cleanup function
}
```

### Infinite Scroll

```javascript
class InfiniteScroll {
  #observer;
  #sentinel;
  #loading = false;

  constructor(container, loadMore) {
    this.#sentinel = document.createElement("div");
    this.#sentinel.className = "scroll-sentinel";
    container.appendChild(this.#sentinel);

    this.#observer = new IntersectionObserver(
      async ([entry]) => {
        if (!entry.isIntersecting || this.#loading) return;

        this.#loading = true;
        try {
          const hasMore = await loadMore();
          if (!hasMore) this.#observer.disconnect(); // no more pages
        } finally {
          this.#loading = false;
        }
      },
      { threshold: 0, rootMargin: "300px" }, // load 300px before sentinel is visible
    );

    this.#observer.observe(this.#sentinel);
  }

  destroy() {
    this.#observer.disconnect();
    this.#sentinel.remove();
  }
}
```

### Sticky Header Detection

```javascript
// Detect when header becomes sticky (without scroll event)
function watchStickyHeader(header) {
  // Place a sentinel element just above the header
  const sentinel = document.createElement("div");
  sentinel.style.cssText = "height: 1px; margin-top: -1px;";
  header.parentNode.insertBefore(sentinel, header);

  const observer = new IntersectionObserver(
    ([entry]) => {
      // When sentinel leaves viewport, header is sticky
      header.classList.toggle("is-sticky", !entry.isIntersecting);
    },
    { threshold: [1] }, // fire when fully in/out of viewport
  );

  observer.observe(sentinel);
  return () => {
    observer.disconnect();
    sentinel.remove();
  };
}
```

---

## 10. ResizeObserver

Fires when an element's size changes — without polling or window resize events. Handles both element-level size changes and CSS grid/flexbox layout changes.

```javascript
const observer = new ResizeObserver((entries) => {
  for (const entry of entries) {
    // entry.target: the observed element
    // entry.contentRect: { width, height, top, left, right, bottom }
    // entry.borderBoxSize: [{ inlineSize, blockSize }] — including borders
    // entry.contentBoxSize: [{ inlineSize, blockSize }] — content area only
    // entry.devicePixelContentBoxSize: pixel-accurate size (for canvas)

    const { width, height } = entry.contentRect;
    console.log(`${entry.target.id}: ${width}px × ${height}px`);
  }
});

observer.observe(element);
observer.unobserve(element);
observer.disconnect();
```

### Auto-Resize Canvas

```javascript
function makeCanvasResponsive(canvas) {
  const ctx = canvas.getContext("2d");

  const observer = new ResizeObserver(([entry]) => {
    // Use devicePixelContentBoxSize for pixel-perfect rendering
    const [{ inlineSize: width, blockSize: height }] =
      entry.devicePixelContentBoxSize ?? [entry.contentBoxSize[0]];

    canvas.width = width;
    canvas.height = height;

    // Re-render after resize
    redrawCanvas(ctx, width, height);
  });

  observer.observe(canvas, {
    box: "device-pixel-content-box", // request pixel-accurate measurements
  });

  return () => observer.disconnect();
}
```

### Component That Adapts to Its Container Size

```javascript
class AdaptiveComponent {
  #element;
  #observer;

  constructor(element) {
    this.#element = element;

    this.#observer = new ResizeObserver(([entry]) => {
      const { width } = entry.contentRect;
      this.#adaptToSize(width);
    });

    this.#observer.observe(element);
  }

  #adaptToSize(width) {
    const el = this.#element;
    // Apply different layouts based on available width
    el.classList.toggle("layout-compact", width < 300);
    el.classList.toggle("layout-medium", width >= 300 && width < 600);
    el.classList.toggle("layout-expanded", width >= 600);
  }

  destroy() {
    this.#observer.disconnect();
  }
}
```

---

## 11. PerformanceObserver

Observes performance timeline entries — long tasks, resource timings, paint timings, layout shifts.

```javascript
// Observe long tasks (tasks > 50ms — causes jank)
const longTaskObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn(
      `Long task: ${entry.duration.toFixed(1)}ms`,
      `\nAttribution:`,
      entry.attribution,
    );

    // Report to monitoring
    reportMetric("longTask", {
      duration: entry.duration,
      startTime: entry.startTime,
    });
  }
});

longTaskObserver.observe({ entryTypes: ["longtask"] });

// Observe Core Web Vitals
const cwvObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    switch (entry.entryType) {
      case "largest-contentful-paint":
        console.log("LCP:", entry.startTime.toFixed(0), "ms");
        break;
      case "first-input":
        console.log("FID:", entry.processingStart - entry.startTime, "ms");
        break;
      case "layout-shift":
        if (!entry.hadRecentInput) {
          console.log("CLS contribution:", entry.value.toFixed(4));
        }
        break;
    }
  }
});

cwvObserver.observe({
  entryTypes: ["largest-contentful-paint", "first-input", "layout-shift"],
});
```

---

## 12. Observable Pattern (RxJS-style)

An Observable represents a sequence of values over time. Unlike a Promise (one value), an Observable can emit many values and can be cancelled.

```javascript
/**
 * Minimal Observable implementation (RxJS-compatible interface).
 */
class Observable {
  #subscribeFn;

  constructor(subscribeFn) {
    this.#subscribeFn = subscribeFn;
  }

  subscribe(observerOrNext, errorFn, completeFn) {
    // Normalize to observer object
    const observer =
      typeof observerOrNext === "function"
        ? {
            next: observerOrNext,
            error: errorFn ?? (() => {}),
            complete: completeFn ?? (() => {}),
          }
        : {
            next: () => {},
            error: () => {},
            complete: () => {},
            ...observerOrNext,
          };

    // Run subscribe function, get cleanup function
    const cleanup = this.#subscribeFn(observer) ?? (() => {});

    // Return subscription with unsubscribe
    return { unsubscribe: cleanup };
  }

  // ── Operators ────────────────────────────────────────────────────

  map(transformFn) {
    return new Observable(
      (observer) =>
        this.subscribe({
          next: (v) => observer.next(transformFn(v)),
          error: (e) => observer.error(e),
          complete: () => observer.complete(),
        }).unsubscribe,
    );
  }

  filter(predicateFn) {
    return new Observable(
      (observer) =>
        this.subscribe({
          next: (v) => predicateFn(v) && observer.next(v),
          error: (e) => observer.error(e),
          complete: () => observer.complete(),
        }).unsubscribe,
    );
  }

  debounceTime(ms) {
    return new Observable((observer) => {
      let timerId;
      const sub = this.subscribe({
        next: (v) => {
          clearTimeout(timerId);
          timerId = setTimeout(() => observer.next(v), ms);
        },
        error: (e) => observer.error(e),
        complete: () => {
          clearTimeout(timerId);
          observer.complete();
        },
      });
      return () => {
        sub.unsubscribe();
        clearTimeout(timerId);
      };
    });
  }

  take(count) {
    return new Observable((observer) => {
      let taken = 0;
      const sub = this.subscribe({
        next: (v) => {
          observer.next(v);
          if (++taken >= count) {
            observer.complete();
            sub.unsubscribe();
          }
        },
        error: (e) => observer.error(e),
        complete: () => observer.complete(),
      });
      return sub.unsubscribe;
    });
  }

  // ── Static Creators ──────────────────────────────────────────────

  static fromEvent(target, eventName) {
    return new Observable((observer) => {
      const handler = (e) => observer.next(e);
      target.addEventListener(eventName, handler);
      return () => target.removeEventListener(eventName, handler);
    });
  }

  static interval(ms) {
    return new Observable((observer) => {
      let count = 0;
      const id = setInterval(() => observer.next(count++), ms);
      return () => clearInterval(id);
    });
  }

  static fromPromise(promise) {
    return new Observable((observer) => {
      promise
        .then((v) => {
          observer.next(v);
          observer.complete();
        })
        .catch((e) => observer.error(e));
    });
  }
}

// Usage
const searchInput = document.getElementById("search");

const search$ = Observable.fromEvent(searchInput, "input")
  .map((e) => e.target.value)
  .filter((v) => v.length >= 2)
  .debounceTime(300);

const subscription = search$.subscribe(async (query) => {
  const results = await fetchSearchResults(query);
  renderResults(results);
});

// Cleanup when done
subscription.unsubscribe();
```

---

## 13. Observer vs Pub/Sub — The Difference

These terms are often used interchangeably but refer to different topologies:

```
OBSERVER PATTERN:
  Subject ──direct reference──► Observer
  Subject knows about observers (holds references)
  Observer knows about subject (registered with it)
  Coupling: moderate (both sides know about each other)

  subject.subscribe(observer);
  subject.notify(data); // directly calls observer.update(data)

PUB/SUB PATTERN:
  Publisher ──► Event Bus ◄── Subscriber
  Publisher knows about the bus (not subscribers)
  Subscriber knows about the bus (not publishers)
  Coupling: loose (neither side knows about the other)

  bus.publish('user:login', data);   // no knowledge of who's listening
  bus.subscribe('user:login', fn);   // no knowledge of who's publishing
```

```javascript
// Observer: subject holds direct references to observers
class Counter {
  #observers = [];
  subscribe(obs) {
    this.#observers.push(obs);
  }
  increment() {
    this.count++;
    this.#observers.forEach((o) => o.onCountChange(this.count)); // direct call
  }
}

// Pub/Sub: intermediary bus — publisher and subscriber decoupled
class EventBus {
  #channels = new Map();

  publish(channel, data) {
    this.#channels.get(channel)?.forEach((fn) => fn(data));
  }

  subscribe(channel, fn) {
    if (!this.#channels.has(channel)) this.#channels.set(channel, new Set());
    this.#channels.get(channel).add(fn);
    return () => this.#channels.get(channel)?.delete(fn);
  }
}

const bus = new EventBus();

// Counter doesn't know about listeners
bus.publish("counter:change", newCount);

// Logger doesn't know about counter
bus.subscribe("counter:change", (count) => console.log("Count:", count));
```

---

## 14. Memory Management in Observer Systems

Observer systems are the #1 source of memory leaks in frontend applications. The pattern is always the same: subscribe but never unsubscribe.

### The Leak Pattern

```javascript
// ❌ Classic leak: component subscribes but never cleans up
class Dashboard {
  constructor() {
    // These subscriptions keep `this` alive forever
    store.subscribe((state) => this.render(state));
    eventBus.on("data:update", (data) => this.processData(data));
    window.addEventListener("resize", () => this.handleResize());

    // When Dashboard is "destroyed" and removed from the page:
    // - store still holds reference to the callback (which closes over `this`)
    // - eventBus still holds reference to the callback
    // - window still holds reference to the callback
    // → Dashboard instance CANNOT be garbage collected
    // → All Dashboard data stays in memory
  }
}
```

### Memory-Safe Observer Pattern

```javascript
// ✅ Collect all cleanup functions, call them on destroy
class Dashboard {
  #cleanups = [];

  constructor() {
    this.#cleanups.push(store.subscribe((state) => this.render(state)));
    this.#cleanups.push(
      eventBus.on("data:update", (data) => this.processData(data)),
    );

    const onResize = () => this.handleResize();
    window.addEventListener("resize", onResize);
    this.#cleanups.push(() => window.removeEventListener("resize", onResize));

    // Native observers
    const ro = new ResizeObserver((entries) => this.onResize(entries));
    ro.observe(this.element);
    this.#cleanups.push(() => ro.disconnect());

    const io = new IntersectionObserver((entries) => this.onIntersect(entries));
    io.observe(this.element);
    this.#cleanups.push(() => io.disconnect());
  }

  destroy() {
    this.#cleanups.forEach((fn) => fn());
    this.#cleanups = [];
  }
}
```

### Use `WeakMap` for Observer Metadata

```javascript
// Attach observer metadata to DOM elements without preventing GC
const elementObservers = new WeakMap();

function observeElement(element, config) {
  const observer = new MutationObserver(config.callback);
  observer.observe(element, config.options);

  // When element is GC'd, WeakMap entry is automatically removed
  elementObservers.set(element, observer);
}

function stopObserving(element) {
  const observer = elementObservers.get(element);
  observer?.disconnect();
  elementObservers.delete(element);
}
```

---

## 15. Good Practices

### ✅ Always return unsubscribe functions

```javascript
// ✅ Every subscription returns its own cleanup
function createSubscription(subject, observer) {
  subject.subscribe(observer);
  return () => subject.unsubscribe(observer); // always return this
}
```

### ✅ Debounce high-frequency observer callbacks

```javascript
// ✅ Don't process every single resize event
const observer = new ResizeObserver((entries) => {
  // Batch processing with rAF — at most once per frame
  if (!this._rafPending) {
    this._rafPending = true;
    requestAnimationFrame(() => {
      this._rafPending = false;
      this.processResize(entries);
    });
  }
});
```

### ✅ Use `once` for single-fire observations

```javascript
// ✅ Unobserve after first intersection
const observer = new IntersectionObserver(([entry]) => {
  if (entry.isIntersecting) {
    loadContent(entry.target);
    observer.unobserve(entry.target); // ← stop observing once loaded
  }
});
```

### ✅ Limit mutation observer scope

```javascript
// ✅ Observe minimum required scope
observer.observe(specificContainer, {
  childList: true,
  subtree: false, // direct children only
  // Only add attributes: true if you actually need attribute changes
});
```

---

## 16. Bad Practices

### ❌ Global EventEmitter with no cleanup

```javascript
// ❌ Module-level emitter — handlers accumulate forever
const globalEmitter = new EventEmitter();

function MyComponent() {
  globalEmitter.on("theme:change", updateTheme); // never removed
  // Each component instance adds another handler
  // After 100 component mount/unmount cycles: 100 handlers
}
```

### ❌ Anonymous functions in subscriptions (can't unsubscribe)

```javascript
// ❌ Can't remove — no reference to the function
subject.subscribe((data) => updateUI(data));

// ✅ Store reference to remove it
const handler = (data) => updateUI(data);
subject.subscribe(handler);
// Later:
subject.unsubscribe(handler);
```

### ❌ Synchronous mutations inside MutationObserver

```javascript
// ❌ Can cause infinite loops — mutation triggers observer
// which triggers mutation which triggers observer...
const observer = new MutationObserver(() => {
  element.setAttribute("data-updated", Date.now()); // triggers observer again!
});

observer.observe(element, { attributes: true });
```

### ❌ IntersectionObserver for layout measurements

```javascript
// ❌ IO gives you intersection data, not precise layout metrics
const io = new IntersectionObserver(([entry]) => {
  const height = entry.boundingClientRect.height; // imprecise for layout work
  positionTooltip(height); // use ResizeObserver instead
});

// ✅ ResizeObserver for size, IntersectionObserver for visibility
```

---

## 17. Common Mistakes

### Mistake 1 — Forgetting that MutationObserver callbacks are batched

```javascript
// MutationObserver batches mutations and fires the callback
// with ALL mutations since the last callback as an array
// (not once per mutation)

const observer = new MutationObserver((mutations) => {
  // mutations is an ARRAY — may have many entries
  // ❌ Don't assume mutations[0] is the only change
  // ✅ Iterate all entries
  for (const mutation of mutations) {
    handleMutation(mutation);
  }
});
```

### Mistake 2 — Not checking `isIntersecting` in IntersectionObserver

```javascript
// IO fires on BOTH enter AND exit
// ❌ Assuming callback only fires on enter
const io = new IntersectionObserver(([entry]) => {
  loadImage(entry.target); // runs on exit too!
});

// ✅ Check isIntersecting
const io = new IntersectionObserver(([entry]) => {
  if (entry.isIntersecting) loadImage(entry.target);
});
```

### Mistake 3 — Creating observers inside loops without cleanup

```javascript
// ❌ Creates N observers that are never disconnected
items.forEach((item) => {
  const observer = new ResizeObserver(handleResize);
  observer.observe(item); // no reference kept → can't disconnect
});

// ✅ One observer for all items
const observer = new ResizeObserver((entries) => {
  entries.forEach((entry) => handleResize(entry.target, entry.contentRect));
});

items.forEach((item) => observer.observe(item));
// One observer.disconnect() cleans up all
```

---

## 18. Interview-Level Explanation

> **"What is the Observer pattern? How is it different from Pub/Sub? What browser APIs use it?"**

**Strong answer:**

> "The Observer pattern defines a one-to-many relationship where a subject (the observable) maintains a list of observers and notifies them all when its state changes. The key feature is loose coupling — the subject doesn't need to know what the observers will do with the notification, and observers can be added or removed at runtime.
>
> It differs from Pub/Sub in topology. In the Observer pattern, the subject holds direct references to observers — both sides know about each other. In Pub/Sub, there's a central message broker (event bus) in between — publishers and subscribers have no knowledge of each other, only of the bus. Pub/Sub is more loosely coupled, better for cross-module communication, while Observer is simpler and more efficient for direct relationships.
>
> The browser provides four native observer APIs that follow this pattern: MutationObserver watches for DOM tree changes and is far more efficient than polling for DOM modifications; IntersectionObserver detects when an element enters or leaves the viewport, replacing scroll event listeners for lazy loading and infinite scroll; ResizeObserver fires when an element's size changes, replacing window resize listeners for component-level responsiveness; and PerformanceObserver monitors performance metrics like long tasks, Core Web Vitals, and resource timings.
>
> The most critical production concern with observer systems is memory management. Every subscription creates a reference that prevents garbage collection. The pattern for safe observers is: always return an unsubscribe function from every `subscribe` call, and always call those functions in a component's destroy or unmount lifecycle. The classic leak is a component that subscribes to a global event bus or store in its constructor but never unsubscribes when it's removed — the bus holds a reference to the callback, the callback closes over the component, and the component (plus all its data) never gets garbage collected."

---

## 19. Exercises

### Exercise 1 — Build a DOM event Observable

Create `Observable.fromEvent(element, eventName)` that:

- Returns an Observable of DOM events
- Properly removes the listener when unsubscribed
- Works with the debounceTime operator for search-as-you-type

```javascript
const searchQuery$ = Observable.fromEvent(input, "input")
  .map((e) => e.target.value)
  .filter((q) => q.length >= 2)
  .debounceTime(300);

const sub = searchQuery$.subscribe((query) => fetchResults(query));
// Later:
sub.unsubscribe(); // listener removed from DOM
```

<details>
<summary>Solution</summary>

```javascript
static fromEvent(target, eventName) {
  return new Observable(observer => {
    const handler = (event) => observer.next(event);
    target.addEventListener(eventName, handler);
    // Return cleanup function — called on unsubscribe
    return () => target.removeEventListener(eventName, handler);
  });
}
```

The full Observable class from Section 12 includes `map`, `filter`, and `debounceTime` — combine them with `fromEvent` and the implementation is complete.

</details>

---

### Exercise 2 — Memory-safe component base class

Build a `Component` base class that:

- Collects all subscriptions in its constructor
- Automatically cleans up on `destroy()`
- Provides helpers: `watch(store, path)`, `listen(emitter, event)`, `observe(element)` for ResizeObserver

<details>
<summary>Solution</summary>

```javascript
class Component {
  #cleanups = [];

  // Subscribe to a store path
  watch(store, path, handler) {
    this.#cleanups.push(store.watch(path, handler));
  }

  // Listen to an EventEmitter
  listen(emitter, event, handler) {
    this.#cleanups.push(emitter.on(event, handler));
  }

  // Observe element size with ResizeObserver
  observeResize(element, handler) {
    const ro = new ResizeObserver((entries) => handler(entries[0]));
    ro.observe(element);
    this.#cleanups.push(() => ro.disconnect());
  }

  // Observe element visibility
  observeIntersection(element, handler, options = {}) {
    const io = new IntersectionObserver(([entry]) => handler(entry), options);
    io.observe(element);
    this.#cleanups.push(() => io.disconnect());
  }

  // Register any custom cleanup
  onCleanup(fn) {
    this.#cleanups.push(fn);
  }

  // Call in component destroy/unmount
  destroy() {
    this.#cleanups.forEach((fn) => fn());
    this.#cleanups = [];
  }
}

// Usage
class UserCard extends Component {
  constructor(element, userId) {
    super();
    this.element = element;

    // All registered — auto-cleaned on destroy()
    this.watch(userStore, `users.${userId}`, (user) => this.render(user));
    this.listen(appEvents, "theme:change", (theme) => this.applyTheme(theme));
    this.observeResize(element, (entry) =>
      this.onResize(entry.contentRect.width),
    );
  }

  render(user) {
    /* ... */
  }
  applyTheme(theme) {
    /* ... */
  }
  onResize(width) {
    /* ... */
  }
  // destroy() inherited — cleans up everything
}
```

</details>

---

### Exercise 3 — Reactive computed values

Using the `ReactiveSystem` from Section 6, implement:

- A todo list with reactive state
- A computed `completedCount` that updates automatically
- A computed `pendingItems` array that filters incomplete todos
- An effect that updates the DOM when either computed changes

```javascript
const rs = new ReactiveSystem();
const state = rs.reactive({
  todos: [
    { id: 1, text: "Learn Proxies", done: true },
    { id: 2, text: "Build reactive system", done: false },
    { id: 3, text: "Ship it", done: false },
  ],
});

const completedCount = rs.computed(
  () => state.todos.filter((t) => t.done).length,
);

const pendingItems = rs.computed(() => state.todos.filter((t) => !t.done));

rs.effect(() => {
  document.getElementById("completed").textContent =
    `${completedCount.value} of ${state.todos.length} done`;
});

// Test: mark a todo done
state.todos[1].done = true;
// DOM should automatically update: "2 of 3 done"
```

---

## 🔗 Related Topics

- [`javascript-core/15-pub-sub-systems.md`](./15-pub-sub-systems.md) — Pub/Sub architecture patterns
- [`javascript-core/05-closures.md`](./05-closures.md) — Closures in observer callbacks
- [`javascript-core/08-memory-management.md`](./08-memory-management.md) — Observer memory leaks
- [`patterns/01-observer.md`](../patterns/01-observer.md) — Observer design pattern catalog entry
- [`performance/09-intersection-observer.md`](../performance/09-intersection-observer.md) — IntersectionObserver for performance

---

<div align="center">

**Next:** [`javascript-core/15-pub-sub-systems.md`](./15-pub-sub-systems.md) →

</div>
