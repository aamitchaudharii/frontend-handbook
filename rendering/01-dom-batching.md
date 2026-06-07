# 01 — DOM Batching

> **"Every DOM write that triggers a layout is a negotiation between JavaScript and the browser's rendering engine. Batching is about having fewer, larger negotiations instead of many small ones — and making sure you're not accidentally asking for the result before the negotiation is over."**

DOM batching is the discipline of grouping DOM reads and writes to minimize layout recalculations. The browser's rendering engine is highly optimized, but it can only maintain that optimization if you give it a chance to batch work. Interleaving reads and writes forces synchronous layout recalculation — one of the most expensive operations in the browser's pipeline. This document covers the mechanics of forced layout, batching strategies, the tools that automate batching, and the patterns that make it second nature.

---

## 📚 Table of Contents

1. [Why Batching Matters](#1-why-batching-matters)
2. [The Layout Invalidation Model](#2-the-layout-invalidation-model)
3. [What Forces Synchronous Layout](#3-what-forces-synchronous-layout)
4. [The Read-Write-Read-Write Anti-Pattern](#4-the-read-write-read-write-anti-pattern)
5. [Basic Batching — All Reads, Then All Writes](#5-basic-batching--all-reads-then-all-writes)
6. [requestAnimationFrame Batching](#6-requestanimationframe-batching)
7. [FastDOM — Automatic Batching](#7-fastdom--automatic-batching)
8. [Scheduler API](#8-scheduler-api)
9. [React Automatic Batching](#9-react-automatic-batching)
10. [CSS Containment for Scoped Layouts](#10-css-containment-for-scoped-layouts)
11. [Virtual DOM as a Batching Layer](#11-virtual-dom-as-a-batching-layer)
12. [Measuring Layout Thrashing](#12-measuring-layout-thrashing)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. Why Batching Matters

Layout (reflow) is one of the most expensive browser operations. The browser must:

1. Walk the render tree
2. Compute geometry for every affected element
3. Propagate geometry to dependent elements

```
Cost of one layout recalculation:
  Simple page (< 100 elements): ~0.1ms
  Medium page (500 elements):   ~0.5ms
  Complex page (5,000 elements): ~5ms
  Very complex (table-heavy):   ~20ms+

If you force 100 layouts in a loop:
  Simple page: 100 × 0.1ms = 10ms  (within budget)
  Medium page: 100 × 0.5ms = 50ms  (3 dropped frames)
  Complex page: 100 × 5ms  = 500ms (30 dropped frames = total jank)
```

---

## 2. The Layout Invalidation Model

The browser optimizes layout with a "dirty bit" system:

```
STATE 1: Layout is CLEAN
  All element positions/sizes are up to date.
  Reading layout properties returns cached values instantly.
  Cost: near zero.

STATE 2: Layout is DIRTY
  A DOM mutation changed geometry.
  Browser sets a dirty flag — layout is not yet recalculated.
  Reading layout properties: browser MUST recalculate immediately.
  Cost: proportional to page complexity.

THE TRAP:
  Write: element.style.width = '200px';   → layout becomes DIRTY
  Read:  element.offsetWidth;              → forces synchronous recalculation
  Write: element.style.height = '100px';  → layout becomes DIRTY again
  Read:  element.offsetHeight;            → forces another recalculation

  Each read-after-write pair: one full synchronous layout
```

### The Batching Optimization

```
DIRTY STATE → multiple writes → READ → layout recalculates ONCE

Write: element1.style.width  = '200px'; → dirty
Write: element2.style.height = '100px'; → dirty (still only ONCE pending)
Write: element3.style.left   = '50px';  → dirty
Read:  element1.offsetWidth;            → ONE layout recalculation for all three
```

---

## 3. What Forces Synchronous Layout

A read of any geometry property while layout is dirty forces synchronous recalculation:

```javascript
// GEOMETRY READS (all force layout when dirty):
element.offsetWidth     element.offsetHeight
element.offsetTop       element.offsetLeft
element.offsetParent

element.scrollWidth     element.scrollHeight
element.scrollTop       element.scrollLeft

element.clientWidth     element.clientHeight
element.clientTop       element.clientLeft

element.getBoundingClientRect()
element.getBoundingClientRects() (plural)
element.getClientRects()

window.scrollX          window.scrollY
window.pageXOffset      window.pageYOffset
window.innerWidth       window.innerHeight

document.scrollingElement.scrollTop

// LAYOUT FORCING METHODS:
element.focus()           // forces layout to ensure element is visible
element.scrollIntoView()
element.scrollIntoViewIfNeeded()

// CSS COMPUTED VALUES:
getComputedStyle(element).width   // may force layout
getComputedStyle(element).height

// RANGE METHODS:
range.getBoundingClientRect()
```

---

## 4. The Read-Write-Read-Write Anti-Pattern

This is the classic "layout thrashing" pattern:

```javascript
// ❌ Read-Write interleaving: one layout per element
const elements = document.querySelectorAll(".item");

elements.forEach((el) => {
  const width = el.offsetWidth; // READ: forces layout (dirty after prev write)
  el.style.width = width * 1.1 + "px"; // WRITE: invalidates layout

  const height = el.offsetHeight; // READ: forces layout AGAIN (dirty after write above)
  el.style.height = height * 1.1 + "px"; // WRITE: invalidates layout
});

// For 100 elements: 200 layout recalculations
// For a complex page: 200 × 5ms = 1,000ms = ~60 dropped frames
```

```javascript
// ✅ Batch reads then writes: one layout total
const elements = document.querySelectorAll(".item");

// Phase 1: ALL reads (one layout recalculation at most)
const dimensions = Array.from(elements).map((el) => ({
  el,
  width: el.offsetWidth,
  height: el.offsetHeight,
}));

// Phase 2: ALL writes (layout invalidated once, recalculated once at next read/paint)
dimensions.forEach(({ el, width, height }) => {
  el.style.width = width * 1.1 + "px";
  el.style.height = height * 1.1 + "px";
});

// Total: 1 layout recalculation
```

---

## 5. Basic Batching — All Reads, Then All Writes

### Manual Batch Pattern

```javascript
// Pattern: collect all reads, then apply all writes

function measureAndResize(elements) {
  // STEP 1: Read all measurements (triggers at most one layout)
  const measurements = elements.map((el) => ({
    element: el,
    rect: el.getBoundingClientRect(),
    computed: window.getComputedStyle(el),
  }));

  // STEP 2: Compute new values (pure JS, no DOM reads)
  const updates = measurements.map(({ element, rect, computed }) => ({
    element,
    newWidth: `${rect.width * 1.1}px`,
    newHeight: `${rect.height * 1.1}px`,
    newColor: computed.color === "rgb(0, 0, 0)" ? "#333" : "black",
  }));

  // STEP 3: Write all updates (one batch of writes)
  updates.forEach(({ element, newWidth, newHeight, newColor }) => {
    element.style.width = newWidth;
    element.style.height = newHeight;
    element.style.color = newColor;
  });
}
```

### Separating Read and Write Functions

```javascript
// Architectural pattern: separate read functions from write functions

// ✅ Write function: never reads, only writes
function applyStyles(element, styles) {
  Object.entries(styles).forEach(([prop, value]) => {
    element.style[prop] = value;
  });
}

// ✅ Read function: only reads, never writes
function measureElement(element) {
  return {
    width: element.offsetWidth,
    height: element.offsetHeight,
    rect: element.getBoundingClientRect(),
  };
}

// ✅ Orchestrator: reads first, then writes
function updateLayout(elements) {
  const measurements = elements.map(measureElement); // all reads
  measurements.forEach((m, i) => {
    // all writes
    applyStyles(elements[i], computeNewStyles(m));
  });
}
```

---

## 6. requestAnimationFrame Batching

rAF provides a natural batching boundary — all writes from a rAF callback are applied together before the next paint.

```javascript
class DOMBatcher {
  #reads = [];
  #writes = [];
  #rafId = null;

  read(fn) {
    this.#reads.push(fn);
    this.#scheduleFlush();
    return this;
  }

  write(fn) {
    this.#writes.push(fn);
    this.#scheduleFlush();
    return this;
  }

  #scheduleFlush() {
    if (!this.#rafId) {
      this.#rafId = requestAnimationFrame(() => this.#flush());
    }
  }

  #flush() {
    this.#rafId = null;

    // 1. Execute all reads (flush any pending layouts first)
    const reads = this.#reads.splice(0);
    reads.forEach((fn) => {
      try {
        fn();
      } catch (err) {
        console.error("Batcher read error:", err);
      }
    });

    // 2. Execute all writes (batched — only one layout triggered at next read)
    const writes = this.#writes.splice(0);
    writes.forEach((fn) => {
      try {
        fn();
      } catch (err) {
        console.error("Batcher write error:", err);
      }
    });
  }
}

const batcher = new DOMBatcher();

// Usage — schedule reads and writes, they execute in the correct order
elements.forEach((el) => {
  let measurement;

  batcher.read(() => {
    measurement = el.getBoundingClientRect(); // collected with other reads
  });

  batcher.write(() => {
    el.style.transform = `translateX(${measurement.width / 2}px)`; // batched with other writes
  });
});
```

---

## 7. FastDOM — Automatic Batching

FastDOM is a library that provides a clean API for scheduling DOM reads and writes on the next animation frame.

```javascript
// Install: npm install fastdom
import fastdom from "fastdom";

// ✅ FastDOM: automatic batching via rAF
function updateAllElements(elements) {
  elements.forEach((el) => {
    let width;

    // Schedule a read
    fastdom.measure(() => {
      width = el.offsetWidth; // collected with all other measure() calls
    });

    // Schedule a write (runs after all measure() calls complete)
    fastdom.mutate(() => {
      el.style.width = width * 1.1 + "px"; // batched with all other mutate() calls
    });
  });
}

// FastDOM guarantees:
// 1. All measure() callbacks run before any mutate() callbacks
// 2. Within a single rAF: measures first, then mutations
// 3. If a mutate schedules a measure, it runs in the NEXT rAF (clean separation)

// With Promise API:
async function getWidthAndResize(element) {
  const width = await fastdom.measure(() => element.offsetWidth);
  await fastdom.mutate(() => {
    element.style.width = width * 2 + "px";
  });
}

// Clearing scheduled tasks:
const task = fastdom.measure(() => expensiveRead(el));
fastdom.clear(task); // cancel if no longer needed
```

---

## 8. Scheduler API

The Scheduler API (available in Chrome as `scheduler.postTask()`) provides fine-grained control over task priority:

```javascript
// Scheduler API (progressive enhancement — not yet universal)
if ("scheduler" in window) {
  // Schedule non-visual work at lower priority
  scheduler.postTask(
    () => {
      updateAnalyticsDashboard();
    },
    { priority: "background" },
  ); // doesn't compete with rendering

  // Schedule visual updates at high priority
  scheduler.postTask(
    () => {
      applyUserInputToDOM();
    },
    { priority: "user-blocking" },
  ); // runs before rendering

  // Schedule medium-priority work
  scheduler.postTask(
    () => {
      prefetchNextPageData();
    },
    { priority: "user-visible" },
  );
}

// Polyfill for older browsers: map to setTimeout/rAF
const scheduleTask = window.scheduler?.postTask
  ? (fn, opts) => scheduler.postTask(fn, opts)
  : (fn, opts) => {
      if (opts?.priority === "user-blocking") return queueMicrotask(fn);
      if (opts?.priority === "background") return setTimeout(fn, 0);
      return requestAnimationFrame(fn);
    };
```

### `scheduler.yield()` — Cooperative Scheduling

```javascript
// Yield control back to the browser mid-task to allow rendering
async function processLargeList(items) {
  for (let i = 0; i < items.length; i++) {
    processItem(items[i]);

    // Yield every 50 items to allow rendering and input handling
    if (i % 50 === 0) {
      if ("scheduler" in window && scheduler.yield) {
        await scheduler.yield(); // Chrome 115+
      } else {
        await new Promise((resolve) => setTimeout(resolve, 0));
      }
    }
  }
}
```

---

## 9. React Automatic Batching

React 18 introduced automatic batching — multiple state updates in any asynchronous context are batched into a single render.

### Pre-React 18 (Manual Batching Required)

```javascript
// React 17 and earlier: state updates in async contexts caused multiple renders
setTimeout(() => {
  setCount((c) => c + 1); // render 1
  setFlag((f) => !f); // render 2
  // Two separate renders!
}, 1000);

// React 17 workaround: unstable_batchedUpdates
import { unstable_batchedUpdates } from "react-dom";

setTimeout(() => {
  unstable_batchedUpdates(() => {
    setCount((c) => c + 1); // batched
    setFlag((f) => !f); // batched
    // Single render
  });
}, 1000);
```

### React 18 Automatic Batching

```javascript
// React 18: automatic batching everywhere
setTimeout(() => {
  setCount((c) => c + 1); // ┐
  setFlag((f) => !f); // ┘ batched automatically → single render
}, 1000);

Promise.resolve().then(() => {
  setCount((c) => c + 1); // ┐
  setFlag((f) => !f); // ┘ batched → single render
});

fetch("/api/data").then((data) => {
  setData(data); // ┐
  setLoading(false); // ┘ batched → single render
});

// Works in ALL async contexts in React 18
```

### Opting Out of Batching

```javascript
// If you genuinely need separate renders:
import { flushSync } from "react-dom";

flushSync(() => {
  setCount((c) => c + 1); // forces immediate render
});
// DOM is updated synchronously here

flushSync(() => {
  setFlag((f) => !f); // forces another immediate render
});

// Use sparingly — defeats the optimization
// Valid use case: measuring DOM after state change
flushSync(() => setOpen(true));
const height = dropdownRef.current.offsetHeight; // safe to read now
```

---

## 10. CSS Containment for Scoped Layouts

CSS `contain` limits the scope of layout recalculations to the contained element and its descendants.

```css
/* contain: layout — layout changes inside don't affect outside */
.card {
  contain: layout;
  /* Changes to children cannot affect siblings or ancestors */
  /* Reduces scope of layout recalculation */
}

/* contain: strict — most aggressive containment */
.widget {
  contain: strict;
  /* Equivalent to: contain: size layout style paint */
  /* No layout or paint effects can escape this element */
  /* Requires explicit dimensions */
  width: 300px;
  height: 200px;
}

/* contain: content — size is exempt (can still expand) */
.feed-item {
  contain: content;
  /* Equivalent to: contain: layout paint style */
  /* Overflow is hidden */
}
```

```javascript
// With CSS containment: layout recalculation is scoped
// Without containment: changing one element triggers global layout check
// With contain: layout: browser checks only the contained subtree

// Benefit: even without explicit batching, each contained update
// costs proportional to the contained subtree, not the whole page

// Best for: repeated UI elements (list items, cards, widgets)
.list-item { contain: content; }
// 1000 list items: each update only recalculates within the item
```

---

## 11. Virtual DOM as a Batching Layer

React's virtual DOM acts as an automatic batching layer — state changes accumulate into a virtual diff, which is then applied to the real DOM in a single pass.

```
Without virtual DOM:
  setState('Alice') → DOM update immediately
  setState('Bob')   → DOM update immediately
  2 DOM mutations, 2 layout triggers

With virtual DOM (React):
  setState('Alice') → virtual DOM updated (no real DOM change)
  setState('Bob')   → virtual DOM updated (no real DOM change)
  React batches: computes one diff, applies one set of DOM mutations
  1 DOM mutation batch, potentially 1 layout trigger
```

```javascript
// React batches multiple state updates
function handleClick() {
  setName("Bob"); // virtual DOM update
  setAge(25); // virtual DOM update
  setActive(true); // virtual DOM update
  // React processes all three in one reconciliation pass
  // Real DOM gets one batch of mutations
}

// React also batches across components in the same event handler:
function handleSubmit(e) {
  e.preventDefault();
  formStore.setLoading(true); // component A updates
  userStore.setUser(formData); // component B updates
  routerStore.navigate("/done"); // component C updates
  // All three components re-render in one batch
}
```

---

## 12. Measuring Layout Thrashing

### Chrome DevTools — Performance Panel

```
Warning signs in the Performance panel:
  Purple "Recalculate Style" blocks per frame — expensive
  Interleaved "Layout" events — layout thrashing
  "Forced reflow" warnings in the timeline

Forced reflow indicator:
  Performance panel → hover over a Layout event
  "Forced reflow is a likely performance bottleneck"
  The call stack shows WHICH code forced the layout
```

### Programmatic Detection

```javascript
// Wrap layout-forcing reads to detect and log them
const originalOffsetWidth = Object.getOwnPropertyDescriptor(
  HTMLElement.prototype,
  "offsetWidth",
);

if (process.env.NODE_ENV === "development") {
  Object.defineProperty(HTMLElement.prototype, "offsetWidth", {
    get() {
      // Track layout reads in development
      if (window.__layoutReadTracker) {
        window.__layoutReadTracker.count++;
        window.__layoutReadTracker.stack = new Error().stack;
      }
      return originalOffsetWidth.get.call(this);
    },
  });
}

// Usage:
window.__layoutReadTracker = { count: 0, stack: null };
performSuspiciousOperation();
console.log(`Layout reads: ${window.__layoutReadTracker.count}`);
```

### Performance Observer for Long Tasks

```javascript
// Detect long tasks (> 50ms) that may include layout thrashing
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      console.warn(`Long task: ${entry.duration.toFixed(0)}ms`);
      // If this correlates with DOM operations: likely layout thrashing
    }
  }
});
observer.observe({ entryTypes: ["longtask"] });
```

---

## 13. Good Practices

### ✅ Use CSS classes instead of individual style properties

```javascript
// ✅ One class change = one style recalculation
element.classList.add("is-expanded");
// CSS handles multiple property changes atomically

// ❌ Multiple style changes = potential multiple recalculations
element.style.height = "200px";
element.style.padding = "16px";
element.style.overflow = "hidden";
```

### ✅ Use CSS transforms instead of layout-triggering properties

```javascript
// ❌ Triggers layout
element.style.left = "100px";
element.style.top = "50px";

// ✅ Compositor-only — no layout trigger
element.style.transform = "translate(100px, 50px)";
```

### ✅ Batch DOM reads before writes in animation frames

```javascript
// ✅ Always: reads → writes within one rAF callback
requestAnimationFrame(() => {
  // Phase 1: reads
  const positions = elements.map((el) => el.getBoundingClientRect());

  // Phase 2: writes
  elements.forEach((el, i) => {
    el.style.transform = computeTransform(positions[i]);
  });
});
```

### ✅ Apply CSS `contain` to repeated elements

```css
/* ✅ Limit layout scope for list items */
.card,
.list-row,
.feed-item {
  contain: content; /* or layout */
}
```

---

## 14. Bad Practices

### ❌ Reading `offsetWidth` inside a loop after writing

```javascript
// ❌ Classic layout thrashing in a loop
const items = document.querySelectorAll(".item");
items.forEach((item) => {
  item.classList.add("expanded"); // write: layout dirty
  const height = item.offsetHeight; // read: forced sync layout
  item.style.marginBottom = height + "px"; // write: layout dirty again
});
// N reads → N layouts
```

### ❌ Calling `getComputedStyle` after mutations

```javascript
// ❌ getComputedStyle forces layout after any write
element.style.display = "block"; // write
const color = getComputedStyle(element).color; // read: forced layout
// Often unintentional — developers don't realize they read after writing
```

### ❌ Using `scrollTop` getter/setter in animation loops

```javascript
// ❌ Reading scrollTop forces layout if dirty
requestAnimationFrame(function tick() {
  element.style.top = "200px"; // write: dirty
  const scrollPos = element.scrollTop; // read: forced layout
  // ...
  requestAnimationFrame(tick);
});
// Fixes: cache scrollTop at read-phase start, don't read after writing
```

---

## 15. Common Mistakes

### Mistake 1 — Not knowing `getBoundingClientRect` forces layout

```javascript
// ❌ Many developers don't know this forces layout
function isVisible(element) {
  const rect = element.getBoundingClientRect(); // FORCES LAYOUT if dirty!
  return rect.top >= 0 && rect.bottom <= window.innerHeight;
}

// Called 100 times after DOM changes: 100 layouts
elements.forEach(el => {
  el.classList.add('modified'); // write
  if (isVisible(el)) { ... }    // read: forced layout
});

// ✅ Batch: read visibility for all first, then apply changes
const visible = elements.map(el => isVisible(el)); // one layout pass
elements.forEach((el, i) => {
  if (visible[i]) el.classList.add('modified');     // write phase
});
```

### Mistake 2 — `el.style.cssText` still triggers layout on read

```javascript
// ❌ Misconception: cssText avoids layout forcing
// cssText is still a write — reading geometry after it forces layout
element.style.cssText = "width: 100px; height: 50px;"; // write
const w = element.offsetWidth; // still forces layout (cssText was a write)

// ✅ The rule is simple: any layout-forcing read after any write = forced layout
// Batch all reads before any writes
```

### Mistake 3 — ResizeObserver callbacks not batched with other writes

```javascript
// ❌ Writing to DOM inside ResizeObserver can cause ResizeObserver loops
const ro = new ResizeObserver((entries) => {
  entries.forEach((entry) => {
    const { width } = entry.contentRect;
    entry.target.style.fontSize = `${width / 10}px`; // write
    // This triggers resize → calls observer again → infinite loop
  });
});

// ✅ Only read in ResizeObserver, schedule writes separately
const ro = new ResizeObserver((entries) => {
  entries.forEach((entry) => {
    const { width } = entry.contentRect;
    requestAnimationFrame(() => {
      entry.target.style.fontSize = `${width / 10}px`; // deferred write
    });
  });
});
```

---

## 16. Interview-Level Explanation

> **"What is layout thrashing and how do you prevent it?"**

**Strong answer:**

> "Layout thrashing occurs when JavaScript forces the browser to perform synchronous layout recalculations multiple times in a single frame. The browser optimizes layout by batching DOM mutations and only recalculating geometry when it has to — typically once per frame before paint. But if you read a layout property like `offsetWidth` or `getBoundingClientRect` after writing to the DOM, the browser must recalculate layout synchronously to return an accurate value. If you then write more, you've invalidated the layout again. The next read forces another recalculation. This read-write-read-write pattern can cause dozens or hundreds of layout recalculations per frame, each costing several milliseconds on a complex page.
>
> The solution is to batch reads and writes separately. You do all layout-forcing reads first — they can share the single up-to-date layout. Then you do all writes. The writes invalidate layout once. By the time the next read happens (if any), it's in the next task or rAF callback, giving the browser a chance to batch the work.
>
> Practically, there are a few approaches. Manual batching: separate your code into explicit read and write phases. Libraries like FastDOM automate this — you schedule reads and writes, and it ensures they run in the correct order within the animation frame. React's virtual DOM is an automatic batching layer: state changes accumulate into a virtual diff, which is applied to the real DOM in one pass. React 18's automatic batching extends this to async contexts.
>
> CSS containment (`contain: layout`) limits the scope of layout recalculations — changes inside a contained element can't affect layout outside it, which reduces the cost of any single layout trigger.
>
> To detect layout thrashing: Chrome DevTools Performance panel shows 'Forced reflow' markers on Layout events, which include the call stack that triggered it. PerformanceObserver with `entryTypes: ['longtask']` catches long tasks that often contain thrashing."

---

## 17. Exercises

### Exercise 1 — Fix layout thrashing

```javascript
// ❌ This function has severe layout thrashing for large lists. Fix it.
function equalizeHeights(container) {
  const items = container.querySelectorAll(".item");

  items.forEach((item) => {
    // Find the tallest sibling
    let maxHeight = 0;
    items.forEach((sibling) => {
      const h = sibling.offsetHeight; // forces layout per sibling!
      if (h > maxHeight) maxHeight = h;
    });

    item.style.height = maxHeight + "px"; // write: dirty
  });
}

// For 100 items: 100 × 100 = 10,000 layout reads!
```

<details>
<summary>Solution</summary>

```javascript
// ✅ Fixed: single read pass, then single write pass

function equalizeHeights(container) {
  const items = Array.from(container.querySelectorAll(".item"));

  // PHASE 1: Read all heights (one layout pass for all)
  const heights = items.map((item) => item.offsetHeight);

  // Compute max height (pure JS — no DOM access)
  const maxHeight = Math.max(...heights);

  // PHASE 2: Write all heights (one layout invalidation)
  items.forEach((item) => {
    item.style.height = maxHeight + "px";
  });

  // Total layout recalculations: 1 (down from 10,000)
}

// Even better with rAF batching:
function equalizeHeightsAnimated(container) {
  const items = Array.from(container.querySelectorAll(".item"));

  requestAnimationFrame(() => {
    // READS: all heights in one pass
    const maxHeight = Math.max(...items.map((i) => i.offsetHeight));

    // WRITES: all applied in same frame before paint
    items.forEach((i) => {
      i.style.height = maxHeight + "px";
    });
  });
}
```

</details>

---

### Exercise 2 — Implement a read-write batcher

Build a `DOMBatch` class that:

- Accepts `.read(fn)` and `.write(fn)` calls at any time
- Executes all reads before all writes, within the next animation frame
- Returns a Promise from each call that resolves when the fn has run

```typescript
const batch = new DOMBatch();

const widthPromise = batch.read(() => element.offsetWidth);
const writePromise = batch.write(() => {
  element.style.color = "red";
});

const width = await widthPromise; // resolves after the rAF read phase
await writePromise; // resolves after the rAF write phase
```

<details>
<summary>Solution</summary>

```typescript
class DOMBatch {
  #reads: Array<{
    fn: () => unknown;
    resolve: (v: unknown) => void;
    reject: (e: unknown) => void;
  }> = [];
  #writes: Array<{
    fn: () => unknown;
    resolve: (v: unknown) => void;
    reject: (e: unknown) => void;
  }> = [];
  #rafId: number | null = null;

  read<T>(fn: () => T): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      this.#reads.push({
        fn,
        resolve: resolve as (v: unknown) => void,
        reject,
      });
      this.#schedule();
    });
  }

  write<T>(fn: () => T): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      this.#writes.push({
        fn,
        resolve: resolve as (v: unknown) => void,
        reject,
      });
      this.#schedule();
    });
  }

  #schedule() {
    if (!this.#rafId) {
      this.#rafId = requestAnimationFrame(() => this.#flush());
    }
  }

  #flush() {
    this.#rafId = null;

    // Execute all reads
    const reads = this.#reads.splice(0);
    for (const { fn, resolve, reject } of reads) {
      try {
        resolve(fn());
      } catch (err) {
        reject(err);
      }
    }

    // Execute all writes
    const writes = this.#writes.splice(0);
    for (const { fn, resolve, reject } of writes) {
      try {
        resolve(fn());
      } catch (err) {
        reject(err);
      }
    }
  }
}
```

</details>

---

## 🔗 Related Topics

- [`performance/03-layout-thrashing.md`](../performance/03-layout-thrashing.md) — Layout thrashing in depth
- [`performance/04-raf-optimization.md`](../performance/04-raf-optimization.md) — rAF for batching animation work
- [`browser-internals/04-layout-reflow.md`](../browser-internals/04-layout-reflow.md) — How layout works
- [`browser-internals/06-composite-layers.md`](../browser-internals/06-composite-layers.md) — Compositor-only properties that avoid layout

---

<div align="center">

**Next:** [`rendering/02-virtual-dom.md`](./02-virtual-dom.md) →

</div>
