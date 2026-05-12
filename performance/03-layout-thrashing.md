# 03 — Layout Thrashing

> **"Layout thrashing is when you force the browser to do in one frame what it was designed to spread across many. It's the most common cause of janky UIs in production — and it's almost always invisible until you profile."**

Layout thrashing can turn a smooth 60fps interface into a slideshow. It happens silently, it scales badly, and it's found in virtually every large frontend codebase that hasn't been deliberately audited. This document covers the mechanism, the patterns, the fixes, and how to detect it in production code.

---

## 📚 Table of Contents

1. [What Is Layout Thrashing?](#1-what-is-layout-thrashing)
2. [Why the Browser Batches Layout](#2-why-the-browser-batches-layout)
3. [The Invalidation-Read Cycle](#3-the-invalidation-read-cycle)
4. [Properties That Force Layout](#4-properties-that-force-layout)
5. [Classic Thrashing Patterns](#5-classic-thrashing-patterns)
6. [Measuring the Damage](#6-measuring-the-damage)
7. [Fix Strategy 1 — Separate Reads and Writes](#7-fix-strategy-1--separate-reads-and-writes)
8. [Fix Strategy 2 — FastDOM Pattern](#8-fix-strategy-2--fastdom-pattern)
9. [Fix Strategy 3 — RAF Batching](#9-fix-strategy-3--raf-batching)
10. [Fix Strategy 4 — CSS Solutions](#10-fix-strategy-4--css-solutions)
11. [Fix Strategy 5 — CSS Containment](#11-fix-strategy-5--css-containment)
12. [Real-World Case Studies](#12-real-world-case-studies)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Detection Tools & Workflow](#15-detection-tools--workflow)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. What Is Layout Thrashing?

Layout thrashing (also called **forced synchronous layout**) happens when JavaScript:

1. **Writes** to the DOM (invalidates layout)
2. **Reads** a layout property (forces the browser to recalculate layout immediately)
3. **Writes** again (invalidates layout again)
4. **Reads** again (forces another recalculation)
5. ...repeat, often inside a loop

Each forced recalculation is a full layout pass — the browser recomputes the geometry of the entire document (or a significant subtree). Do this 100 times in a single frame and you've turned a 2ms layout into a 200ms layout.

### The Analogy

Imagine a chef who:

- Gets one order, goes to the kitchen, cooks it, comes back
- Gets the next order, goes to the kitchen, cooks it, comes back
- ...for 100 orders

vs. a chef who:

- Takes all 100 orders first
- Goes to the kitchen once
- Cooks everything in one session
- Delivers everything

The browser's layout engine is the kitchen. It works best when you batch.

---

## 2. Why the Browser Batches Layout

The browser doesn't recalculate layout every time you change a style. That would be catastrophically slow. Instead, it **marks layout as dirty** (invalidated) and queues the recalculation for later — specifically, just before the next paint.

```javascript
// What you write:
element.style.width = "200px"; // Layout marked DIRTY
element.style.height = "100px"; // Layout still DIRTY (not recalculated again)
element.style.margin = "10px"; // Layout still DIRTY

// Browser's internal state after these three lines:
// "Layout is dirty. I'll recalculate before the next paint."
// Cost: ~0ms — just a flag set

// Then, before the next frame renders:
// Browser recalculates layout ONCE
// Cost: ~2ms
```

This batching is a fundamental browser optimization. Three style changes = one layout recalculation. This is free performance — as long as you don't break the batch.

### What Breaks the Batch

Reading any layout-dependent property after a write forces an immediate recalculation:

```javascript
element.style.width = "200px"; // marks layout dirty
const h = element.offsetHeight; // forces layout NOW — batch broken
// Layout was recalculated after just ONE write
// You've paid the full layout cost prematurely
```

The browser has no choice. You're asking for a value that depends on the current layout state. If it doesn't recalculate now, it would return a stale value. So it recalculates immediately, synchronously, on the main thread.

---

## 3. The Invalidation-Read Cycle

Here's exactly what happens, step by step, during thrashing:

```
Frame starts (16.67ms budget)
│
├── element.style.width = '200px'
│   └── Layout INVALIDATED (dirty flag set)
│
├── const h = element.offsetHeight    ← READ
│   ├── Browser: "layout is dirty, must recalculate NOW"
│   ├── Full layout pass: ~2ms
│   └── Returns correct value
│
├── element.style.height = h + 'px'
│   └── Layout INVALIDATED again
│
├── const w = element.offsetWidth     ← READ
│   ├── Browser: "layout is dirty, must recalculate NOW"
│   ├── Full layout pass: ~2ms
│   └── Returns correct value
│
├── ... (repeat 100x in a loop)
│
└── Total layout time: 200ms
    Frame budget exceeded by 12x
    Result: browser skips ~11 frames
    User sees: frozen UI for 200ms
```

Contrast with the normal (batched) flow:

```
Frame starts (16.67ms budget)
│
├── element.style.width = '200px'   ← WRITE
├── element.style.height = '100px'  ← WRITE
├── element.style.margin = '10px'   ← WRITE
│
└── (browser recalculates layout once before paint)
    Layout pass: ~2ms
    Total: 2ms instead of 200ms
```

---

## 4. Properties That Force Layout

Memorize this list. Reading any of these after a style write forces synchronous layout.

### Geometry Properties

```javascript
// Element dimensions
element.offsetWidth;
element.offsetHeight;
element.offsetTop;
element.offsetLeft;
element.offsetParent;

// Client dimensions (padding included, border excluded)
element.clientWidth;
element.clientHeight;
element.clientTop;
element.clientLeft;

// Scroll dimensions
element.scrollWidth;
element.scrollHeight;
element.scrollTop; // WRITE also forces layout (scroll position change)
element.scrollLeft;

// Bounding rect (most commonly used — most commonly abused)
element.getBoundingClientRect();
element.getClientRects();

// Focus
element.focus(); // yes, focus forces layout
```

### Window & Document Properties

```javascript
window.innerWidth;
window.innerHeight;
window.scrollX;
window.scrollY;
document.documentElement.offsetHeight;
document.documentElement.scrollTop;
```

### Computed Styles

```javascript
// Forces CSSOM + layout if any pending mutations
window.getComputedStyle(element);
window.getComputedStyle(element, "::before");
```

### Methods That Force Layout

```javascript
element.getBoundingClientRect(); // most common cause
element.getClientRects();
element.scrollIntoView();
element.scrollIntoViewIfNeeded();
range.getBoundingClientRect();
range.getClientRects();
```

> **Full list:** The definitive reference is maintained by Paul Irish (Chrome DevRel) and covers every layout-triggering property across browsers.

---

## 5. Classic Thrashing Patterns

### Pattern 1 — The Loop

The most common and most damaging pattern:

```javascript
// THRASHING — 1 forced layout per element
const items = document.querySelectorAll(".item"); // 500 items

items.forEach((item) => {
  const width = item.offsetWidth; // READ: forces layout
  item.style.width = width * 1.1 + "px"; // WRITE: invalidates layout
  // Next iteration: READ again → forces layout again
  // 500 items = 500 forced layouts
});

// With 500 elements and ~2ms per layout: 1000ms of layout work in one frame.
```

```javascript
// FIXED — 1 layout total
const items = Array.from(document.querySelectorAll(".item"));

// Phase 1: ALL reads
const widths = items.map((item) => item.offsetWidth); // 1 forced layout total

// Phase 2: ALL writes
items.forEach((item, i) => {
  item.style.width = widths[i] * 1.1 + "px"; // no layout triggered
});
// Browser does 1 final layout before next paint
```

### Pattern 2 — Resize Handler Without Batching

```javascript
// THRASHING — fires ~60x/second during resize
window.addEventListener("resize", () => {
  const containerWidth = container.offsetWidth; // READ: forced layout
  children.forEach((child) => {
    child.style.width = containerWidth / children.length + "px"; // WRITE
    const childHeight = child.offsetHeight; // READ inside write loop: thrashing
    child.style.minHeight = childHeight + "px"; // WRITE
  });
});
```

```javascript
// FIXED: debounce + batch reads/writes
let rafId = null;

window.addEventListener("resize", () => {
  if (rafId) cancelAnimationFrame(rafId);
  rafId = requestAnimationFrame(() => {
    // All reads
    const containerWidth = container.offsetWidth;
    const childHeights = Array.from(children).map((c) => c.offsetHeight);

    // All writes
    children.forEach((child, i) => {
      child.style.width = containerWidth / children.length + "px";
      child.style.minHeight = childHeights[i] + "px";
    });
  });
});
```

### Pattern 3 — Animation Loop Without Batching

```javascript
// THRASHING in animation loop — jank every frame
function animate() {
  elements.forEach((el) => {
    const rect = el.getBoundingClientRect(); // READ: forced layout
    const newX = computeNewPosition(rect);
    el.style.left = newX + "px"; // WRITE: layout invalidated
    // Loop continues: next element reads → forced layout again
  });
  requestAnimationFrame(animate);
}
```

```javascript
// FIXED: batch reads before RAF, writes in RAF, use transform not left
function animate() {
  // Read phase — snapshot all positions
  const rects = elements.map((el) => el.getBoundingClientRect());

  requestAnimationFrame(() => {
    // Write phase — use transform (composite only, no layout at all)
    elements.forEach((el, i) => {
      const newX = computeNewPosition(rects[i]);
      el.style.transform = `translateX(${newX}px)`;
    });
  });
}
```

### Pattern 4 — Hidden Thrashing in Third-Party Code

This one is insidious because you don't write the thrashing code directly:

```javascript
// Some UI libraries do this internally:
tooltip.setPosition(anchor); // internally: getBoundingClientRect + style mutation
tooltip.show(); // internally: display change + getBoundingClientRect
popover.alignTo(trigger); // internally: reads + writes

// If called in sequence without batching:
// Each call forces layout after the previous write
```

**Fix:** Use libraries that support deferred/batched positioning (like Floating UI), or batch multiple position updates yourself with the FastDOM pattern.

### Pattern 5 — Scroll Event + DOM Reads

```javascript
// THRASHING — fires on every scroll pixel
window.addEventListener("scroll", () => {
  const scrollY = window.scrollY; // OK — doesn't force layout
  const rect = header.getBoundingClientRect(); // READ: forces layout
  if (rect.bottom < 0) {
    header.classList.add("sticky"); // WRITE: invalidates layout
  }
});
// On fast scroll: 60+ events/second × forced layout = disaster
```

```javascript
// FIXED: Use IntersectionObserver — browser-native, no layout thrashing
const observer = new IntersectionObserver(
  ([entry]) => {
    header.classList.toggle("sticky", !entry.isIntersecting);
  },
  { threshold: 0 },
);

observer.observe(headerSentinel); // observe a sentinel element at top of content
```

---

## 6. Measuring the Damage

### Chrome DevTools — Identifying Thrashing

In the Performance tab, layout thrashing shows as a characteristic pattern:

```
Timeline (zoomed in to one frame):

THRASHING:
│ Script   ████████████████████████████████
│ Layout   ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██  ← many small layouts interleaved
│ Script   ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██

HEALTHY (batched):
│ Script   ████████████████████████████████
│ Layout                                  ███  ← one layout at the end
```

**Warning signs to look for:**

- Many small purple (layout) bars interleaved with yellow (script) bars
- "Forced reflow" label in the call stack tooltip
- Long frames (> 16ms) even for simple-seeming operations
- Flame graph showing Layout called multiple times within one Task

### Programmatic Benchmarking

```javascript
// Measure thrashing vs batched — run in your browser console
function benchmark(label, fn, iterations = 3) {
  const times = [];
  for (let i = 0; i < iterations; i++) {
    const start = performance.now();
    fn();
    times.push(performance.now() - start);
  }
  const avg = times.reduce((a, b) => a + b, 0) / times.length;
  console.log(
    `${label}: avg ${avg.toFixed(2)}ms (runs: ${times.map((t) => t.toFixed(1)).join(", ")}ms)`,
  );
}

// Setup: 200 divs
const container = document.createElement("div");
document.body.appendChild(container);
for (let i = 0; i < 200; i++) {
  const el = document.createElement("div");
  el.style.cssText = "width:100px;height:20px;background:#eee;margin:2px;";
  container.appendChild(el);
}
const elements = Array.from(container.querySelectorAll("div"));

// Test thrashing
benchmark("Thrashing (200 elements)", () => {
  elements.forEach((el) => {
    el.style.width = el.offsetWidth + 5 + "px"; // READ + WRITE per element
  });
  elements.forEach((el) => (el.style.width = "100px")); // reset
});

// Test batched
benchmark("Batched (200 elements)", () => {
  const widths = elements.map((el) => el.offsetWidth); // all reads
  elements.forEach((el, i) => {
    el.style.width = widths[i] + 5 + "px"; // all writes
  });
  elements.forEach((el) => (el.style.width = "100px")); // reset
});

// Typical results:
// Thrashing: avg 95.40ms
// Batched:   avg 2.10ms
// Speedup: ~45x
```

---

## 7. Fix Strategy 1 — Separate Reads and Writes

The simplest and most effective fix. Structure your code so all reads happen before any writes in a given operation.

```javascript
// BEFORE: interleaved reads and writes
function layoutPanels(panels) {
  panels.forEach((panel) => {
    const headerH = panel.querySelector(".header").offsetHeight; // READ
    const footerH = panel.querySelector(".footer").offsetHeight; // READ
    const totalH = panel.offsetHeight; // READ
    const body = panel.querySelector(".body");
    body.style.height = totalH - headerH - footerH + "px"; // WRITE
    body.style.marginTop = headerH + "px"; // WRITE
  });
}
// n panels = n * 3 forced layouts
```

```javascript
// AFTER: all reads, then all writes
function layoutPanels(panels) {
  // --- READ PHASE ---
  const measurements = Array.from(panels).map((panel) => ({
    panel,
    body: panel.querySelector(".body"),
    headerH: panel.querySelector(".header").offsetHeight,
    footerH: panel.querySelector(".footer").offsetHeight,
    totalH: panel.offsetHeight,
  }));

  // --- WRITE PHASE ---
  measurements.forEach(({ body, headerH, footerH, totalH }) => {
    body.style.height = totalH - headerH - footerH + "px";
    body.style.marginTop = headerH + "px";
  });
  // 1 layout total regardless of panel count
}
```

### When Reads and Writes Can't Be Separated

Sometimes you genuinely need the result of a write before you can read. Use RAF to separate across frames:

```javascript
// Pattern: write → render → read → write
// Use case: animate element from 0 to its natural height
function animateExpand(element) {
  // Frame 1: set to auto to discover natural height
  element.style.height = "auto";

  requestAnimationFrame(() => {
    // Frame 2: read after browser laid out the auto height
    const naturalHeight = element.offsetHeight;

    // Immediately reset — browser hasn't painted yet
    element.style.height = "0px";
    element.style.transition = "height 300ms ease";

    // Force a style recalculation before starting animation
    element.offsetHeight; // intentional read to flush styles

    requestAnimationFrame(() => {
      // Frame 3: trigger the animated expansion
      element.style.height = naturalHeight + "px";
    });
  });
}
```

---

## 8. Fix Strategy 2 — FastDOM Pattern

For larger codebases where reads and writes are scattered across many functions and modules, use a centralized read/write queue.

```javascript
/**
 * FastDOM — batch reads and writes to prevent layout thrashing
 * Reads are always flushed before writes within each RAF cycle
 */
class FastDOM {
  constructor() {
    this._reads = [];
    this._writes = [];
    this._rafId = null;
  }

  /** Schedule a DOM measurement (read). Runs before all writes. */
  measure(fn) {
    this._reads.push(fn);
    this._scheduleFlush();
    return this;
  }

  /** Schedule a DOM mutation (write). Runs after all reads. */
  mutate(fn) {
    this._writes.push(fn);
    this._scheduleFlush();
    return this;
  }

  /** Cancel all pending work */
  clear() {
    if (this._rafId) cancelAnimationFrame(this._rafId);
    this._rafId = null;
    this._reads = [];
    this._writes = [];
  }

  _scheduleFlush() {
    if (!this._rafId) {
      this._rafId = requestAnimationFrame(() => this._flush());
    }
  }

  _flush() {
    this._rafId = null;

    const reads = this._reads.splice(0);
    const writes = this._writes.splice(0);

    // ALL reads first
    reads.forEach((fn) => {
      try {
        fn();
      } catch (e) {
        console.error("[FastDOM] read error", e);
      }
    });

    // THEN all writes
    writes.forEach((fn) => {
      try {
        fn();
      } catch (e) {
        console.error("[FastDOM] write error", e);
      }
    });

    // Flush any tasks queued during this cycle
    if (this._reads.length || this._writes.length) {
      this._scheduleFlush();
    }
  }
}

export const fastDOM = new FastDOM();
```

**Usage across multiple modules — all automatically batched together:**

```javascript
// module-a.js
import { fastDOM } from "./fastdom.js";

function checkOverflow(element) {
  let isOverflowing;
  fastDOM.measure(() => {
    isOverflowing = element.scrollWidth > element.clientWidth;
  });
  fastDOM.mutate(() => {
    element.classList.toggle("truncated", isOverflowing);
  });
}

// module-b.js
import { fastDOM } from "./fastdom.js";

function positionTooltip(tooltip, anchor) {
  let rect;
  fastDOM.measure(() => {
    rect = anchor.getBoundingClientRect();
  });
  fastDOM.mutate(() => {
    tooltip.style.transform = `translate(${rect.left}px, ${rect.bottom + 8}px)`;
  });
}

// When both are called in the same frame:
// checkOverflow(el);
// positionTooltip(tip, btn);
//
// Execution order:
// 1. measure: isOverflowing = el.scrollWidth > el.clientWidth  (read)
// 2. measure: rect = anchor.getBoundingClientRect()             (read)
// 3. mutate:  el.classList.toggle(...)                          (write)
// 4. mutate:  tooltip.style.transform = ...                     (write)
// Zero thrashing regardless of how many modules participate
```

---

## 9. Fix Strategy 3 — RAF Batching

For animation loops and continuous updates, `requestAnimationFrame` is the natural write boundary.

```javascript
class AnimationBatcher {
  constructor() {
    this.pendingUpdates = new Map(); // element → update fn (latest wins)
    this.rafId = null;
  }

  /** Queue an update. If the same element is updated multiple times
   *  before the next frame, only the last update runs. */
  update(element, updateFn) {
    this.pendingUpdates.set(element, updateFn);
    if (!this.rafId) {
      this.rafId = requestAnimationFrame(() => this._flush());
    }
  }

  _flush() {
    this.rafId = null;
    this.pendingUpdates.forEach((fn, el) => fn(el));
    this.pendingUpdates.clear();
  }
}

const batcher = new AnimationBatcher();

// Multiple scroll handlers — only one RAF callback fires
function onScroll() {
  const scrollY = window.scrollY; // single read before queueing writes

  batcher.update(header, (el) => {
    el.style.transform = `translateY(${Math.min(0, -scrollY)}px)`;
  });
  batcher.update(progressBar, (el) => {
    const progress =
      scrollY / (document.body.scrollHeight - window.innerHeight);
    el.style.transform = `scaleX(${progress})`;
  });
  batcher.update(backToTop, (el) => {
    el.style.opacity = scrollY > 500 ? "1" : "0";
  });
}
```

---

## 10. Fix Strategy 4 — CSS Solutions

Sometimes the best fix for layout thrashing is to avoid triggering layout at all.

### Move positions with `transform` instead of `top`/`left`

```css
/* BEFORE: triggers layout per frame */
.card {
  transition:
    left 300ms,
    top 300ms;
}
.card:hover {
  left: 4px;
  top: -4px;
}

/* AFTER: composite only — no layout, no paint */
.card {
  transition: transform 300ms;
}
.card:hover {
  transform: translate(4px, -4px);
}
```

```javascript
// BEFORE: triggers layout
el.style.left = x + "px";
el.style.top = y + "px";

// AFTER: composite only
el.style.transform = `translate(${x}px, ${y}px)`;
```

### CSS Custom Properties for Dynamic Values

```javascript
// BEFORE: multiple style mutations → multiple style recalculations
element.style.width = computedWidth + "px";
element.style.height = computedHeight + "px";
element.style.margin = computedMargin + "px";

// AFTER: one custom property update → one style recalculation
// CSS handles the rest via variables
document.documentElement.style.setProperty("--panel-w", computedWidth + "px");
document.documentElement.style.setProperty("--panel-h", computedHeight + "px");
document.documentElement.style.setProperty("--panel-m", computedMargin + "px");
```

```css
.panel {
  width: var(--panel-w);
  height: var(--panel-h);
  margin: var(--panel-m);
}
```

---

## 11. Fix Strategy 5 — CSS Containment

CSS `contain` is a powerful, underused tool that limits the scope of layout recalculations. When you mark an element with `contain: layout`, the browser knows that changes inside it cannot affect anything outside it — so it can skip recalculating the rest of the document.

```css
/* Layout containment: internal layout isolated from the outside */
.widget {
  contain: layout;
}

/* Paint containment: painting clipped to border box */
.widget {
  contain: paint;
}

/* Content: layout + paint (most commonly useful) */
.widget {
  contain: content;
}

/* Strict: layout + paint + size (best performance, requires fixed size) */
.widget {
  contain: strict;
}
```

### Practical Use: Dashboard Widgets

```javascript
// Without containment:
// Adding one widget's content → full page layout recalculation

// With containment on each widget:
// Each widget's internal layout is isolated
```

```css
.dashboard-widget {
  contain: content; /* layout + paint isolation */
  /* Changes inside .dashboard-widget cannot affect
     the position of any element outside it */
}
```

### Containment for Virtual Lists

```css
.virtual-list-item {
  contain: strict; /* layout + paint + size */
  height: 48px; /* known fixed height enables size containment */
  /* The browser can skip this item entirely during
     layout passes that don't affect it */
}
```

---

## 12. Real-World Case Studies

### Case Study 1 — Data Table Column Resize

**Problem:** A data table with 200 rows had a column resize feature. Dragging a column header caused severe jank (~2fps).

**Root cause:**

```javascript
// Called on every mousemove event
function onColumnResize(e) {
  const delta = e.clientX - startX;
  columns.forEach((col) => {
    const currentWidth = col.offsetWidth; // READ: forced layout
    col.style.width = currentWidth + delta + "px"; // WRITE: invalidates
  });
  // 200 columns × ~2ms layout = 400ms per mousemove
}
```

**Fix:**

```javascript
let columnWidths = []; // snapshot at drag start

function onDragStart() {
  // Read all widths ONCE before any drag movement
  columnWidths = Array.from(columns).map((col) => col.offsetWidth);
}

function onColumnResize(e) {
  const delta = e.clientX - startX;
  // Pure writes — no forced layout
  columns.forEach((col, i) => {
    col.style.width = columnWidths[i] + delta + "px";
  });
}
```

**Result:** Layout time per drag event: 400ms → 2ms. FPS: 2 → 60.

---

### Case Study 2 — Dashboard Widget Auto-Sizing

**Problem:** A dashboard with 50 widgets that auto-sized to their content froze the page for 3 seconds on load.

**Root cause:**

```javascript
widgets.forEach((widget) => {
  widget.render(); // WRITE
  const height = widget.element.scrollHeight; // READ: forced layout
  widget.setHeight(height); // WRITE: invalidates
  const overflow = widget.element.scrollHeight > widget.maxH; // READ: forced layout
  widget.element.classList.toggle("overflow", overflow); // WRITE
});
// 50 widgets × 2 forced layouts × ~3ms = 300ms
```

**Fix:**

```javascript
// Step 1: render everything
widgets.forEach((w) => w.render());

// Step 2: read everything
const data = widgets.map((w) => ({
  widget: w,
  height: w.element.scrollHeight,
  overflow: w.element.scrollHeight > w.maxH,
}));

// Step 3: write everything
data.forEach(({ widget, height, overflow }) => {
  widget.setHeight(height);
  widget.element.classList.toggle("overflow", overflow);
});
```

**Result:** Load time: 3000ms → 180ms (94% reduction).

---

### Case Study 3 — Infinite Scroll with Image Loading

**Problem:** Images didn't load lazily — they all loaded immediately, causing jank on each batch append.

**Root cause:**

```javascript
function appendItems(items) {
  items.forEach((item) => {
    const el = createItemEl(item);
    container.appendChild(el); // WRITE
    const rect = el.getBoundingClientRect(); // READ: forced layout
    if (rect.top < window.innerHeight) {
      loadImage(el); // triggers more layout
    }
  });
}
```

**Fix:**

```javascript
// Use IntersectionObserver — no forced layout
const imgObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        loadImage(entry.target);
        imgObserver.unobserve(entry.target);
      }
    });
  },
  { rootMargin: "200px" },
); // load 200px before they enter viewport

function appendItems(items) {
  const fragment = document.createDocumentFragment();
  items.forEach((item) => {
    const el = createItemEl(item);
    fragment.appendChild(el); // off-DOM, no layout cost
    imgObserver.observe(el); // async observation, no layout cost
  });
  container.appendChild(fragment); // single DOM mutation
}
```

**Result:** Scroll jank eliminated. Image loading became progressive and smooth.

---

## 13. Good Practices

### ✅ Use the Read/Write comment convention

```javascript
// Makes batching intent explicit and reviewable

// --- READ PHASE ---
const containerW = container.offsetWidth;
const items = Array.from(list.children).map((item) => ({
  el: item,
  height: item.offsetHeight,
  width: item.offsetWidth,
}));

// --- WRITE PHASE ---
container.style.maxWidth = containerW + "px";
items.forEach(({ el, height }) => {
  el.style.minHeight = height + "px";
});
```

### ✅ Audit every loop that touches the DOM

Any `forEach`, `for`, `map` that both reads and writes DOM properties is a thrashing candidate. Make it a code review rule.

```javascript
// Code review flag: DOM read inside a loop that also writes
elements.forEach((el) => {
  const h = el.offsetHeight; // RED FLAG in code review
  el.style.height = h + "px";
});
```

### ✅ Cache layout measurements when the value is stable

```javascript
// If container size only changes on resize, cache it
let cachedWidth = container.offsetWidth;

const ro = new ResizeObserver((entries) => {
  cachedWidth = entries[0].contentRect.width; // update only on actual resize
});
ro.observe(container);

// Use cachedWidth everywhere — zero forced layouts in hot paths
```

### ✅ Prefer `ResizeObserver` over polling

```javascript
// Browser-native, async, no forced layout
const ro = new ResizeObserver((entries) => {
  entries.forEach((entry) => {
    const { width, height } = entry.contentRect;
    handleResize(entry.target, width, height);
    // width/height are already computed — no getBoundingClientRect needed
  });
});
ro.observe(element);
```

---

## 14. Bad Practices

### ❌ Reading layout properties inside animation loops

```javascript
// Forces layout every frame — guaranteed jank
function animate() {
  elements.forEach((el) => {
    const rect = el.getBoundingClientRect(); // READ inside loop
    el.style.transform = `translateX(${computeX(rect)}px)`; // WRITE
  });
  requestAnimationFrame(animate);
}
```

### ❌ Using `offsetWidth` to check visibility

```javascript
// Forces layout just to check visibility — very common mistake
function isVisible(el) {
  return el.offsetWidth > 0 && el.offsetHeight > 0; // forced layout
}

// Better: check styles or use IntersectionObserver
function isVisible(el) {
  const s = getComputedStyle(el);
  return s.display !== "none" && s.visibility !== "hidden" && s.opacity !== "0";
}
```

### ❌ `will-change` on everything to "prevent" thrashing

```javascript
// Completely wrong approach — will-change doesn't prevent thrashing
// and causes GPU memory explosion
document.querySelectorAll("*").forEach((el) => {
  el.style.willChange = "transform"; // DO NOT DO THIS
});
```

### ❌ Synchronous scroll-based animation without caching

```javascript
// Read + write on every scroll event
window.addEventListener("scroll", () => {
  element.style.transform = `translateY(${element.getBoundingClientRect().top}px)`; // read + write
});
```

---

## 15. Detection Tools & Workflow

### Step-by-Step Profiling Workflow

```
1. Open Chrome DevTools → Performance tab
2. Click Record
3. Perform the interaction that feels slow (scroll, resize, click)
4. Stop recording
5. In the flame graph, look for:
   a. Long frames (red border at top of timeline)
   b. Many small purple Layout blocks interleaved with yellow Script blocks
   c. "Forced reflow" tooltip on Layout blocks
6. Click a Layout block → check "Initiator" in the Details panel
   → This shows the exact JS line that triggered forced layout
7. Trace back to the read that caused it
8. Apply the appropriate fix strategy
```

### Rendering Panel Checklist

```
DevTools → ⋮ → More tools → Rendering:

✅ Paint flashing      — green overlay shows areas being repainted
✅ Layout Shift Regions — shows CLS-causing elements
✅ FPS meter           — real-time frame rate overlay
```

### Automated Detection in Development

```javascript
// Detect and warn about offsetWidth/offsetHeight reads in dev mode
if (import.meta.env.DEV) {
  const layoutProps = [
    "offsetWidth",
    "offsetHeight",
    "offsetTop",
    "offsetLeft",
    "clientWidth",
    "clientHeight",
    "scrollWidth",
    "scrollHeight",
  ];

  let isDirty = false;

  // Mark layout as dirty on style writes
  const origSetProperty = CSSStyleDeclaration.prototype.setProperty;
  CSSStyleDeclaration.prototype.setProperty = function (...args) {
    isDirty = true;
    return origSetProperty.apply(this, args);
  };

  // Warn on reads when dirty
  layoutProps.forEach((prop) => {
    const descriptor = Object.getOwnPropertyDescriptor(
      HTMLElement.prototype,
      prop,
    );
    if (!descriptor) return;
    Object.defineProperty(HTMLElement.prototype, prop, {
      get() {
        if (isDirty) {
          console.warn(
            `[Layout Thrash] ${prop} read after style mutation.`,
            new Error().stack,
          );
        }
        return descriptor.get.call(this);
      },
      configurable: true,
    });
  });
}
```

---

## 16. Interview-Level Explanation

> **"What is layout thrashing and how do you fix it?"**

**Strong answer:**

> "Layout thrashing is when JavaScript forces the browser to run layout recalculation multiple times within a single frame, instead of once at the end.
>
> The browser optimizes by batching layout work — it marks layout as dirty when you change styles, but defers the actual recalculation until just before the next paint. The problem is that reading certain DOM properties like `offsetWidth`, `getBoundingClientRect`, or `scrollHeight` requires an up-to-date layout. If layout was invalidated since the last recalculation, the browser must recalculate synchronously right then — before returning your value.
>
> The classic pattern is alternating reads and writes in a loop: write a style, read a measurement, write again, read again. With 100 elements, you might trigger 100 full layout passes in one frame, taking 200ms instead of 2ms.
>
> The primary fix is separating reads from writes: collect all measurements first, then apply all DOM changes. This ensures the browser only recalculates layout once, regardless of how many elements you're working with.
>
> For complex codebases where reads and writes are scattered across modules, the FastDOM pattern — a centralized read/write queue that flushes all reads before all writes in a single RAF callback — prevents thrashing automatically.
>
> At the CSS level, using `transform` instead of `top`/`left` eliminates layout triggering entirely for positional animations — those properties are compositor-only and completely bypass the layout engine.
>
> And for containment scenarios, CSS `contain: layout` tells the browser that an element's internal layout cannot affect anything outside it, limiting the scope of any recalculations that do happen."

---

## 17. Exercises

### Exercise 1 — Spot the thrash

Find all layout thrashing in this function and identify each read/write:

```javascript
function positionDropdownItems(dropdown) {
  const items = dropdown.querySelectorAll(".item");
  const containerRect = dropdown.getBoundingClientRect();

  items.forEach((item, index) => {
    item.style.top = containerRect.top + index * 40 + "px";
    const itemWidth = item.offsetWidth;
    item.style.maxWidth = containerRect.width - itemWidth * 0.1 + "px";
    item.style.opacity = "1";
    const isClipped = item.scrollWidth > item.clientWidth;
    item.title = isClipped ? item.textContent : "";
  });
}
```

<details>
<summary>Analysis + Fixed Version</summary>

**Thrashing points:**

```
Line 3: containerRect = getBoundingClientRect()  — READ (OK, nothing written yet)
Line 6: item.style.top = ...                     — WRITE: layout invalidated
Line 7: item.offsetWidth                         — READ: forced layout (after write above)
Line 8: item.style.maxWidth = ...                — WRITE: layout invalidated
Line 9: item.style.opacity = ...                 — paint only, OK
Line 10: item.scrollWidth                        — READ: forced layout
Line 10: item.clientWidth                        — READ: forced layout
Line 11: item.title = ...                        — no layout
```

Pattern per item: W → R → W → W → R → R → W = 2 forced layouts per item.
With 20 items = 40 forced layouts.

**Fixed version:**

```javascript
function positionDropdownItems(dropdown) {
  const items = Array.from(dropdown.querySelectorAll(".item"));

  // --- READ PHASE ---
  const containerRect = dropdown.getBoundingClientRect();
  const measurements = items.map((item) => ({
    el: item,
    width: item.offsetWidth,
    scrollWidth: item.scrollWidth,
    clientWidth: item.clientWidth,
    text: item.textContent,
  }));

  // --- WRITE PHASE ---
  measurements.forEach(({ el, width, scrollWidth, clientWidth, text }, i) => {
    el.style.top = containerRect.top + i * 40 + "px";
    el.style.maxWidth = containerRect.width - width * 0.1 + "px";
    el.style.opacity = "1";
    el.title = scrollWidth > clientWidth ? text : "";
  });
  // 0 forced layouts (containerRect read before any writes)
}
```

</details>

---

### Exercise 2 — Benchmark it yourself

Paste this in your browser's DevTools console on any page with a DOM:

```javascript
// Create test elements
const wrap = document.createElement("div");
document.body.appendChild(wrap);
for (let i = 0; i < 300; i++) {
  const el = document.createElement("div");
  el.style.cssText = "width:100px;height:20px;background:#ddd;margin:1px;";
  wrap.appendChild(el);
}
const els = Array.from(wrap.children);

// Run both approaches and compare
const t1 = performance.now();
els.forEach((el) => {
  el.style.width = el.offsetWidth + 1 + "px";
});
console.log("Thrashing:", (performance.now() - t1).toFixed(1) + "ms");

els.forEach((el) => {
  el.style.width = "100px";
}); // reset

const t2 = performance.now();
const ws = els.map((el) => el.offsetWidth);
els.forEach((el, i) => {
  el.style.width = ws[i] + 1 + "px";
});
console.log("Batched:", (performance.now() - t2).toFixed(1) + "ms");

wrap.remove();
```

Record the speedup ratio. What device did you run it on? Lower-end devices show more dramatic differences.

---

### Exercise 3 — Refactor a tooltip component

Refactor this tooltip to eliminate layout thrashing:

```javascript
class Tooltip {
  constructor() {
    this.el = document.createElement("div");
    this.el.className = "tooltip";
    document.body.appendChild(this.el);
  }

  show(anchor, text) {
    this.el.textContent = text;
    this.el.style.display = "block"; // WRITE

    const anchorRect = anchor.getBoundingClientRect(); // READ: forced layout
    const tooltipRect = this.el.getBoundingClientRect(); // READ: forced layout
    const vw = window.innerWidth;
    const vh = window.innerHeight;

    let x = anchorRect.left + anchorRect.width / 2 - tooltipRect.width / 2;
    let y = anchorRect.top - tooltipRect.height - 8;

    x = Math.max(8, Math.min(x, vw - tooltipRect.width - 8));
    y = Math.max(8, Math.min(y, vh - tooltipRect.height - 8));

    this.el.style.transform = `translate(${x}px, ${y}px)`; // WRITE
  }
}
```

<details>
<summary>Hint + Solution</summary>

**The problem:** `this.el.style.display = 'block'` writes, then `getBoundingClientRect()` on the tooltip forces layout.

**Solution — use opacity/pointer-events trick to measure invisibly:**

```javascript
class Tooltip {
  constructor() {
    this.el = document.createElement("div");
    this.el.className = "tooltip";
    // Always in DOM but invisible — can be measured without triggering visible layout
    this.el.style.cssText =
      "position:fixed;visibility:hidden;pointer-events:none;";
    document.body.appendChild(this.el);
  }

  show(anchor, text) {
    // Set content while invisible — no forced layout issues
    this.el.textContent = text;
    this.el.style.visibility = "hidden"; // ensure invisible during measure

    // --- READ PHASE (no writes since last read) ---
    const anchorRect = anchor.getBoundingClientRect();
    const tooltipRect = this.el.getBoundingClientRect(); // safe: not dirty
    const vw = window.innerWidth;
    const vh = window.innerHeight;

    // --- COMPUTE ---
    let x = anchorRect.left + anchorRect.width / 2 - tooltipRect.width / 2;
    let y = anchorRect.top - tooltipRect.height - 8;
    x = Math.max(8, Math.min(x, vw - tooltipRect.width - 8));
    y = Math.max(8, Math.min(y, vh - tooltipRect.height - 8));

    // --- WRITE PHASE ---
    this.el.style.transform = `translate(${x}px, ${y}px)`;
    this.el.style.visibility = "visible";
  }

  hide() {
    this.el.style.visibility = "hidden";
  }
}
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md) — Why layout is expensive
- [`browser-internals/04-layout-reflow.md`](../browser-internals/04-layout-reflow.md) — Layout deep dive
- [`rendering/01-dom-batching.md`](../rendering/01-dom-batching.md) — DOM batching patterns
- [`performance/04-raf-optimization.md`](./04-raf-optimization.md) — RAF-based scheduling
- [`performance/09-intersection-observer.md`](./09-intersection-observer.md) — Replace scroll reads
- [`anti-patterns/03-layout-thrashing-causes.md`](../anti-patterns/03-layout-thrashing-causes.md) — Anti-pattern catalog

---

<div align="center">

**Next:** [`performance/05-memory-leaks.md`](./05-memory-leaks.md) →

</div>
