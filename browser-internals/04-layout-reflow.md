# 04 — Layout & Reflow

> **"Layout is the browser computing the answer to one question: given all these elements with all these styles, where exactly does each one sit on the page, and how big is it? That question is harder than it sounds — and the answer changes every time you touch a geometry property."**

Layout (also called Reflow) is the browser stage that transforms styled elements into precise geometry. It computes positions, dimensions, margins, padding, line breaks, flex/grid distributions — everything needed to know _where_ to paint pixels. It's the most expensive stage you can trigger from JavaScript, and it cascades in ways that feel unpredictable until you understand the box model, flow algorithms, and what forces the browser to recalculate.

---

## 📚 Table of Contents

1. [What Layout Computes](#1-what-layout-computes)
2. [The Box Model — The Foundation of Layout](#2-the-box-model--the-foundation-of-layout)
3. [Normal Flow — How Elements Position Themselves](#3-normal-flow--how-elements-position-themselves)
4. [The Layout Algorithms](#4-the-layout-algorithms)
5. [What Triggers Layout](#5-what-triggers-layout)
6. [Layout Scope — What Gets Recalculated](#6-layout-scope--what-gets-recalculated)
7. [Forced Synchronous Layout Revisited](#7-forced-synchronous-layout-revisited)
8. [Layout Thrashing — The Pattern](#8-layout-thrashing--the-pattern)
9. [Incremental Layout](#9-incremental-layout)
10. [Viewport and the Layout Viewport](#10-viewport-and-the-layout-viewport)
11. [Containing Blocks](#11-containing-blocks)
12. [Stacking Contexts](#12-stacking-contexts)
13. [Measuring Layout Performance](#13-measuring-layout-performance)
14. [CSS Containment for Layout Isolation](#14-css-containment-for-layout-isolation)
15. [Good Practices](#15-good-practices)
16. [Bad Practices](#16-bad-practices)
17. [Common Mistakes](#17-common-mistakes)
18. [Interview-Level Explanation](#18-interview-level-explanation)
19. [Exercises](#19-exercises)

---

## 1. What Layout Computes

Layout is the process of computing the **box model geometry** for every element in the render tree. At the end of layout, every element has precise values for:

```
For every element, layout computes:
  ┌─────────────────────────────────────────────┐
  │  Position:                                   │
  │    x, y offset from containing block         │
  │    (top-left corner of margin box)           │
  │                                              │
  │  Dimensions:                                 │
  │    width, height (of content area)           │
  │    padding (top, right, bottom, left)        │
  │    border (top, right, bottom, left)         │
  │    margin (top, right, bottom, left)         │
  │                                              │
  │  Line boxes (for text):                      │
  │    Which words are on which line             │
  │    Height of each line                       │
  │    Baseline alignment                        │
  │                                              │
  │  Overflow boxes:                             │
  │    Does content overflow? In which direction?│
  └─────────────────────────────────────────────┘
```

Layout outputs a **layout tree** (frame tree in Firefox) — a parallel structure to the render tree that maps each visible element to its computed geometry.

---

## 2. The Box Model — The Foundation of Layout

Every element in CSS generates a **box**. The box model defines the space an element occupies.

### The Box Model Layers

```
┌─────────────────────────────────────────────────────────┐  ← margin edge
│                      MARGIN                              │
│  ┌───────────────────────────────────────────────────┐  │  ← border edge
│  │                    BORDER                          │  │
│  │  ┌─────────────────────────────────────────────┐  │  │  ← padding edge
│  │  │                  PADDING                     │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │  ← content edge
│  │  │  │                                       │  │  │  │
│  │  │  │            CONTENT                    │  │  │  │
│  │  │  │          (width × height)             │  │  │  │
│  │  │  │                                       │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### `box-sizing` — Two Models

```css
/* content-box (default): width/height = content box only */
.box {
  box-sizing: content-box;
  width: 200px;
  padding: 20px;
  border: 2px solid;
  /* Total rendered width: 200 + 20+20 + 2+2 = 244px */
  /* width property only controls the CONTENT area */
}

/* border-box (modern default via reset): width/height = border box */
.box {
  box-sizing: border-box;
  width: 200px;
  padding: 20px;
  border: 2px solid;
  /* Total rendered width: 200px (padding and border INSIDE the 200px) */
  /* Content area: 200 - 20-20 - 2-2 = 156px */
}
```

`border-box` is almost universally preferred in modern CSS:

```css
/* Universal reset — apply border-box everywhere */
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

### Margin Collapsing

Adjacent vertical margins collapse into one — the largest margin wins, they don't add:

```css
.a {
  margin-bottom: 20px;
}
.b {
  margin-top: 30px;
}

/* Distance between .a and .b: 30px (not 50px) — margins collapsed */
/* The larger margin (30px) wins */
```

Margins collapse when:

- **Adjacent siblings** with vertical margins touch
- **Parent and first/last child** when no border, padding, or height separates them
- **Empty blocks** with only margins

```html
<!-- Parent-child collapse -->
<div class="parent" style="margin-top: 0; padding-top: 0; border-top: none;">
  <div class="child" style="margin-top: 40px;">
    <!-- child's margin-top leaks out of parent! -->
    <!-- parent behaves as if it has 40px margin-top -->
  </div>
</div>

<!-- Fix: add padding-top or border-top to parent to prevent collapse -->
<div class="parent" style="padding-top: 1px;">
  <div class="child" style="margin-top: 40px;">
    <!-- Now: 1px padding contains the child's margin -->
  </div>
</div>
```

---

## 3. Normal Flow — How Elements Position Themselves

In **normal flow**, elements position themselves according to their formatting context.

### Block Formatting Context (BFC)

Block-level elements (divs, paragraphs, headings) stack **vertically**:

```
Parent (BFC):
  ┌──────────────────────────┐
  │  <div> — 100% width      │  ← takes full width
  ├──────────────────────────┤  ← new line
  │  <p> — 100% width        │  ← takes full width
  ├──────────────────────────┤  ← new line
  │  <h2> — 100% width       │  ← takes full width
  └──────────────────────────┘
```

Block elements:

- Start on a new line
- Stretch to fill the available width
- Generate a new line after themselves

### Inline Formatting Context (IFC)

Inline elements (spans, text, images by default) flow **horizontally** like text:

```
  ┌──────────────────────────────────────────┐
  │ text <span>inline</span> more text and   │ ← wraps when full
  │ continued <em>emphasis</em> on next line │
  └──────────────────────────────────────────┘
```

Inline elements:

- Flow horizontally, left to right (in LTR)
- Wrap to next line when they reach the container edge
- Are constrained by line-height and vertical-align
- Cannot have explicit width/height (padding/margin only apply horizontally)

### Flex and Grid Formatting Contexts

```css
/* Flex formatting context */
.flex-container {
  display: flex; /* children become flex items — new formatting context */
  /* children are laid out according to flex algorithm */
}

/* Grid formatting context */
.grid-container {
  display: grid; /* children become grid items */
  grid-template-columns: repeat(3, 1fr);
  /* children placed in grid cells */
}
```

### Out-of-Flow Positioning

Elements can be taken **out of normal flow**:

```css
/* Absolute positioning: removed from flow, positioned relative to containing block */
.overlay {
  position: absolute;
  top: 0;
  left: 0;
  /* Does NOT affect siblings' layout */
  /* Positioned relative to nearest positioned ancestor */
}

/* Fixed positioning: removed from flow, positioned relative to viewport */
.sticky-header {
  position: fixed;
  top: 0;
  /* Does NOT scroll with page */
}

/* Float: partially out of flow */
.aside {
  float: right;
  /* Removed from normal flow for block layout */
  /* But inline content wraps around it */
}
```

---

## 4. The Layout Algorithms

Different CSS display types use different layout algorithms. Each has different computational characteristics.

### Block Layout Algorithm

```
For each block element:
  1. Determine width:
     - 'auto' → fill available width
     - specific value → use it
     - percentage → resolve relative to containing block

  2. Determine height:
     - 'auto' → computed from content height
     - specific value → use it

  3. Position children:
     - Place each child below the previous
     - Apply margin collapsing

  4. Compute own height from children (if auto)
```

### Flex Layout Algorithm

Flex layout is more complex — it involves a two-pass algorithm:

```
Phase 1: Determine flex item sizes
  1. Resolve flex-basis for each item
  2. Collect items onto flex lines (if flex-wrap)
  3. Resolve flexible lengths (flex-grow, flex-shrink)

Phase 2: Determine flex item positions
  1. Apply justify-content (main axis alignment)
  2. Apply align-items / align-self (cross axis alignment)
  3. Compute item positions
```

### Grid Layout Algorithm

Grid layout is the most computationally complex:

```
1. Define grid tracks (columns + rows)
2. Place items in the grid (auto-placement or explicit placement)
3. Size grid tracks:
   - Fixed: use value directly
   - fr units: distribute remaining space proportionally
   - auto: size to content
   - minmax(): constrain with min and max
4. Compute item positions within tracks
5. Handle implicit tracks (created by overflow items)
```

### Why Layout Cost Varies

```
Block layout:     fastest — sequential, predictable
Inline layout:    moderate — must handle text shaping, line breaking
Flex layout:      moderate — two-pass algorithm
Grid layout:      slower — track sizing algorithm, placement algorithm
Table layout:     slow — requires all content before sizing columns
Positioned:       cheap for the element, but affects stacking context
```

---

## 5. What Triggers Layout

Layout is triggered by any change that could affect the **geometry** of the page.

### CSS Properties That Trigger Layout

```css
/* Changes that ALWAYS trigger layout (change geometry): */
width, min-width, max-width
height, min-height, max-height
padding (any side)
margin (any side)
border-width (any side)
top, right, bottom, left (on positioned elements)
font-size, font-family, font-weight, line-height
display
position
overflow
flex-basis, flex-grow, flex-shrink
grid-template-columns, grid-template-rows
column-count, column-width
writing-mode
```

```css
/* These do NOT trigger layout: */
color
background-color, background-image
box-shadow
border-color (only color, not width)
outline
visibility
opacity
transform
filter
cursor
```

### JavaScript APIs That Trigger Layout

Reading these properties forces an immediate layout calculation:

```javascript
// DIMENSIONS:
element.offsetWidth    element.offsetHeight
element.offsetTop      element.offsetLeft
element.offsetParent

// CLIENT DIMENSIONS (padding-edge):
element.clientWidth    element.clientHeight
element.clientTop      element.clientLeft

// SCROLL:
element.scrollWidth    element.scrollHeight
element.scrollTop      element.scrollLeft

// BOUNDING RECT:
element.getBoundingClientRect()
element.getClientRects()

// WINDOW:
window.innerWidth      window.innerHeight
window.scrollX         window.scrollY

// COMPUTED STYLES (geometric properties):
window.getComputedStyle(el).width
window.getComputedStyle(el).height

// FOCUS (causes layout if element is off-screen):
element.focus()

// SCROLL INTO VIEW:
element.scrollIntoView()
element.scrollIntoViewIfNeeded()
```

---

## 6. Layout Scope — What Gets Recalculated

Not every layout change triggers a full-page recalculation. The browser tries to **limit layout scope** to the minimum necessary.

### Full Layout

Triggered when global structures change:

```javascript
// Full layout triggers:
document.body.appendChild(element);    // DOM insertion at top level
document.body.style.fontSize = '18px'; // root font size change affects everything
window.innerWidth changes              // viewport resize
document.body.innerHTML = '...';       // large DOM change
```

### Subtree Layout

When a change can only affect a subtree:

```javascript
// Only the subtree of container needs re-layout
container.style.width = "500px";
container.appendChild(newChild);
```

The browser analyzes dependencies — if a change inside a `contain: layout` element can't escape, only that element's layout runs.

### Single Element Layout (Uncommon)

With strong containment:

```css
.widget {
  contain: strict; /* layout + paint + size */
  width: 300px; /* fixed size */
}
```

Changing content inside `.widget` only relays out `.widget` internally — its size is fixed, so nothing outside changes.

### The Dirty Bit System

The browser uses **dirty bits** to track which elements need layout:

```
When you change a style:
  - The element is marked "dirty" (needs layout)
  - Its ancestors may also be marked dirty (if they size to content)
  - The browser schedules a layout pass

When layout runs:
  - Only dirty elements (and their dirty subtrees) are recalculated
  - Clean elements are skipped (their geometry is cached)

When you READ a layout property mid-script:
  - Browser must run layout NOW for dirty elements
  - Clears the dirty bits
  - Returns accurate values
```

---

## 7. Forced Synchronous Layout Revisited

We covered layout thrashing in the performance section. Here's the mechanism from the browser's perspective:

```
Normal flow (batched):

JavaScript:    write ─────── write ─────── write
Browser:                                          ← layout runs once (before paint)

Forced synchronous layout:

JavaScript:    write ─── READ ─── write ─── READ
Browser:               ↑ layout         ↑ layout
                  (forced by READ)  (forced by READ)
                  sync, blocking    sync, blocking
```

### The Exact Sequence

```javascript
// Step 1: Write → invalidates layout (dirty bit set)
element.style.width = "100px";

// Step 2: Read → browser must compute layout NOW
const h = element.offsetHeight;
// Browser:
//   1. Sees layout is dirty
//   2. Runs full layout synchronously on main thread
//   3. Returns offsetHeight
//   4. Clears dirty bit

// Step 3: Write → invalidates layout again
element.style.height = h + "px";

// Step 4: Read → forced layout AGAIN
const w = element.offsetWidth;
// Same as step 2 — another full synchronous layout

// Two full layouts in one JavaScript call = layout thrashing
```

### The DevTools Warning

Chrome DevTools shows forced synchronous layout in the Performance timeline:

```
Performance Timeline:
  Script ████████
    Layout (forced) ██  ← triggered by read, marked in red
  Script ████████████
    Layout (forced) ██  ← triggered by another read, marked in red
  Layout ███             ← final layout before paint (normal)
  Paint  █████

The "forced" layouts are the problem. One layout at the end is normal.
```

---

## 8. Layout Thrashing — The Pattern

Full coverage in [`performance/03-layout-thrashing.md`](../performance/03-layout-thrashing.md). Here's the layout-specific view.

### Why It's Catastrophic in Loops

```javascript
// Innocent-looking loop — catastrophic performance
const boxes = document.querySelectorAll(".box");

boxes.forEach((box) => {
  const width = box.offsetWidth; // READ: forced layout
  box.style.width = width * 1.1 + "px"; // WRITE: invalidates
  // Next iteration READ: forced layout again on full document
});

// With 200 boxes and ~2ms per layout: 200 × 2ms = 400ms
// The browser recalculates the ENTIRE document layout 200 times
// All in one JavaScript frame — brutal
```

### Why Each Layout Recalculates the Whole Document

After each `WRITE`, the browser marks the element (and its ancestors) dirty. When you then `READ`, the browser must recalculate all dirty elements. If ancestors are dirty, their subtrees are too. In a typical app, the dirty set quickly expands to the whole document.

```
Write width of .box:
  → .box is dirty
  → .box's parent (flex container) is dirty
  → .box's grandparent is dirty
  → body is dirty
  → html is dirty
  → Full document layout needed

After 200 writes, 200 full document layouts run
```

---

## 9. Incremental Layout

Modern browsers use **incremental layout** to avoid re-laying out the entire document on every change.

### How Incremental Layout Works

```
Incremental layout tracks:
  1. Which elements are "dirty" (need layout)
  2. What changed (size, position, children)
  3. Whether the change can "escape" the element's subtree

If a change is self-contained:
  → Only the element and its subtree are re-laid out

If a change affects ancestors:
  → Walk up the tree marking ancestors dirty too
  → Re-layout the expanded dirty set
```

### What Prevents Incremental Layout

```css
/* Elements that require layout of their entire container: */
float         /* floats affect sibling layout */
absolute      /* still in stacking context, may affect scrollable area */
table-cell    /* cell sizing affects entire table */

/* Elements that propagate changes upward: */
/* Auto-height parents: any change to child height → parent must re-layout */
.parent {
  height: auto; /* sizes to content — always dependent on children */
}
```

### Helping the Browser with Containment

```css
/* contain: layout — block layout effects from escaping */
.widget {
  contain: layout;
  /* Changes inside .widget cannot affect layout outside */
  /* Browser can skip recalculating .widget's siblings/parents */
}

/* contain: size — element size doesn't depend on content */
.card {
  contain: size;
  width: 300px;
  height: 200px;
  /* Fixed size: parent doesn't need to re-layout when content changes */
}
```

---

## 10. Viewport and the Layout Viewport

### What Is the Layout Viewport?

The **layout viewport** is the rectangle the browser uses as the reference for percentage widths, viewport units (`vw`, `vh`), and `position: fixed` placement.

```
Mobile: layout viewport ≠ visual viewport

Layout viewport (what CSS uses):
  └── width: 980px (default on many mobile browsers — causes zoom-out)

Visual viewport (what user sees):
  └── width: 375px (actual device screen width)

This is why you need the viewport meta tag:
<meta name="viewport" content="width=device-width, initial-scale=1">

With this tag:
  Layout viewport = device width (375px)
  Visual viewport = 375px
  No zoom-out, content designed for mobile
```

### Viewport Units and Layout

```css
/* These units reference the layout viewport */
.hero {
  width: 100vw; /* 100% of layout viewport width */
  height: 100vh; /* 100% of layout viewport height */
  /* On mobile: 100vh includes the browser chrome (address bar) */
  /* Element may be taller than the visible area */
}

/* Newer, more predictable units: */
.hero {
  height: 100svh; /* small viewport height: excludes browser chrome */
  height: 100lvh; /* large viewport height: assumes chrome is hidden */
  height: 100dvh; /* dynamic viewport height: updates as chrome shows/hides */
}
```

### Viewport Resize and Layout

```javascript
// Viewport resize triggers a full layout
window.addEventListener("resize", () => {
  // Every element with percentage widths, viewport units, or
  // auto sizing needs layout recalculation
  // This is why resize handlers must be debounced
});

// ✅ ResizeObserver: per-element, no forced layout
const ro = new ResizeObserver((entries) => {
  entries.forEach(({ contentRect }) => {
    // contentRect.width/height provided — no need for offsetWidth
    adjustLayout(contentRect.width);
  });
});
ro.observe(container);
```

---

## 11. Containing Blocks

Every element's geometry is computed relative to its **containing block**. Understanding containing blocks is key to understanding `position`, `%` values, and layout bugs.

### Rules for Containing Block

```
For position: static or relative:
  → Containing block = content area of nearest block ancestor

For position: absolute:
  → Containing block = padding area of nearest POSITIONED ancestor
     (ancestor with position: relative, absolute, fixed, or sticky)
  → If no positioned ancestor: containing block = initial containing block (viewport)

For position: fixed:
  → Containing block = viewport
  → EXCEPTION: if an ancestor has transform, filter, or perspective
    → that ancestor becomes the containing block!

For position: sticky:
  → Behaves like relative until scroll threshold, then like fixed
  → Sticks within its containing block
```

### The Transform Exception

```css
/* GOTCHA: transforms create new containing block for fixed children */
.modal-container {
  transform: translateX(0); /* triggers new containing block */
}

.modal-overlay {
  position: fixed;
  /* EXPECTED: positioned relative to viewport */
  /* ACTUAL: positioned relative to .modal-container! */
  /* Because transform creates a new containing block */
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  /* These will be 100% of .modal-container, not the viewport */
}
```

### Percentage Values and Containing Blocks

```css
.child {
  width: 50%; /* 50% of containing block's WIDTH */
  height: 50%; /* 50% of containing block's HEIGHT */
  /* NOTE: height % only works if parent has explicit height */

  padding-top: 20%; /* 20% of containing block's WIDTH */
  /* (padding percentages always use WIDTH, even vertical) */

  margin-left: 10%; /* 10% of containing block's WIDTH */
}
```

---

## 12. Stacking Contexts

A **stacking context** is a three-dimensional concept — elements within a stacking context are painted as a group before or after other stacking contexts.

### What Creates a Stacking Context

```css
/* These properties create new stacking contexts: */
position: absolute | relative | fixed | sticky + z-index (not auto)
opacity: less than 1
transform (any value other than none)
filter (any value other than none)
will-change: transform | opacity
isolation: isolate
mix-blend-mode (not normal)
clip-path, mask
perspective
contain: layout | paint
```

### Stacking Order Within a Context

```
Painting order (back to front):
  1. Background and borders of stacking context element
  2. Negative z-index descendants (most negative first)
  3. Block-level descendants in flow (normal flow)
  4. Floating descendants
  5. Inline descendants in flow
  6. z-index: 0 or auto descendants
  7. Positive z-index descendants (lowest z-index first)
```

### z-index Only Works Within a Stacking Context

```html
<!-- GOTCHA: z-index comparison is within the same stacking context -->

<div class="container-a" style="position: relative; z-index: 1;">
  <div class="child-a" style="position: relative; z-index: 9999;">
    <!-- z-index: 9999 within container-a -->
  </div>
</div>

<div class="container-b" style="position: relative; z-index: 2;">
  <div class="child-b" style="position: relative; z-index: 1;">
    <!-- z-index: 1 within container-b -->
    <!-- BUT container-b has z-index: 2 vs container-a's z-index: 1 -->
    <!-- So child-b PAINTS ABOVE child-a despite child-a having z-index: 9999! -->
  </div>
</div>

<!-- child-b is above child-a because container-b > container-a in stacking order -->
<!-- z-index of children is ONLY compared within the SAME stacking context -->
```

---

## 13. Measuring Layout Performance

### Chrome DevTools — Layout Profiling

```
Performance tab → Record → interact → Stop

In the flame graph, look for:
  Purple bars = Style + Layout + Paint

"Layout" bar details:
  - Duration: how long layout took
  - "Layout Forced" label: triggered by JS read (bad)
  - Root node: which element was the root of the layout operation
  - Nodes requiring layout: count of elements recalculated

Warning signs:
  - Many small purple bars (thrashing)
  - Large single purple bar (expensive layout)
  - "Layout Forced" labels (forced synchronous layout)
  - Layout time > 5ms per frame
```

### Programmatic Layout Timing

```javascript
// Measure layout time with Performance API
performance.mark("layout-start");

// DOM mutations that trigger layout
container.appendChild(newElement);
container.style.width = "500px";

// Force layout to complete (for measurement)
container.getBoundingClientRect(); // forces layout

performance.mark("layout-end");
performance.measure("layout-duration", "layout-start", "layout-end");

const [measure] = performance.getEntriesByName("layout-duration");
console.log("Layout took:", measure.duration.toFixed(2) + "ms");
```

### Layout Instability — CLS Detection

```javascript
// Detect layout shifts (Cumulative Layout Shift)
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      console.warn("Layout shift:", entry.value.toFixed(4));
      console.log(
        "Sources:",
        entry.sources.map((s) => s.node),
      );
    }
  }
});

observer.observe({ entryTypes: ["layout-shift"] });
```

---

## 14. CSS Containment for Layout Isolation

CSS `contain` is one of the most powerful (and underused) performance tools for layout. It tells the browser that layout effects cannot escape an element.

### `contain: layout`

```css
.widget {
  contain: layout;
  /* PROMISE: no element inside this widget affects layout outside it
     and no layout outside affects layout inside */

  /* Browser can now:
     - Skip re-laying out siblings when widget changes
     - Skip re-laying out .widget when siblings change
     - Optimize layout to only run inside .widget subtree */
}
```

### `contain: size`

```css
.card {
  contain: size;
  width: 300px;
  height: 200px;
  /* PROMISE: this element's size does not depend on its content
     Even if content grows or shrinks, .card stays 300×200px */

  /* This means parent doesn't need to re-layout when content changes */
}
```

### `contain: strict` (Most Aggressive)

```css
.isolated-component {
  contain: strict; /* = layout + paint + size */
  /* Maximum isolation — only changes inside this element affect
     layout, paint, and sizing inside it */
}
```

### Real-World Containment Usage

```css
/* Dashboard widgets — each is fully isolated */
.dashboard-widget {
  contain: content; /* layout + paint */
  /* Widgets can be updated independently */
  /* Changing widget A doesn't trigger layout for widget B */
}

/* Virtual list items — fixed size, fully isolated */
.list-item {
  contain: strict;
  height: 48px;
  /* Each item's content is isolated */
  /* Virtual scroll can recycle items without triggering full list layout */
}

/* Chat messages — layout isolated */
.message {
  contain: layout;
  /* New messages don't trigger layout of older messages */
}
```

---

## 15. Good Practices

### ✅ Separate DOM reads and writes

```javascript
// ✅ All reads first, then all writes
const measurements = elements.map((el) => ({
  el,
  width: el.offsetWidth, // READ phase
  height: el.offsetHeight,
}));

measurements.forEach(({ el, width, height }) => {
  el.style.minWidth = width + "px"; // WRITE phase
  el.style.minHeight = height + "px";
});
// One layout total, regardless of element count
```

### ✅ Use `will-change` for elements that will animate their position

```css
/* ✅ Promotes to compositor layer before animation starts */
.animated-card.will-animate {
  will-change: transform;
}

/* Remove after animation to free GPU memory */
.animated-card {
  will-change: auto;
}
```

### ✅ Use `transform` for positional animations, not `top`/`left`

```css
/* ❌ Triggers layout every frame */
@keyframes slide-in {
  from {
    left: -300px;
  }
  to {
    left: 0;
  }
}

/* ✅ Composite only — no layout, no paint */
@keyframes slide-in {
  from {
    transform: translateX(-300px);
  }
  to {
    transform: translateX(0);
  }
}
```

### ✅ Apply CSS containment to large component trees

```css
/* Any component that updates frequently and independently */
.data-table {
  contain: layout;
}
.chart-widget {
  contain: layout;
}
.notification {
  contain: content;
}
```

### ✅ Use `ResizeObserver` instead of window resize for component sizing

```javascript
// ✅ Fires only when component size changes, provides size without read
const ro = new ResizeObserver(([{ contentRect }]) => {
  adjustComponentLayout(contentRect.width, contentRect.height);
});
ro.observe(component);
```

---

## 16. Bad Practices

### ❌ Reading layout properties in animation loops

```javascript
// ❌ Forces layout every frame — guarantees jank
function animate() {
  const rect = element.getBoundingClientRect(); // forced layout per frame
  updatePosition(rect.x, rect.y);
  requestAnimationFrame(animate);
}
```

### ❌ Setting styles then reading geometry immediately

```javascript
// ❌ Each read after write forces a layout
popup.style.display = "block";
const height = popup.offsetHeight; // forced layout
popup.style.top = `${-height}px`;
```

### ❌ Animating properties that trigger layout

```css
/* ❌ width/height animation = layout every frame */
.expanding {
  animation: expand 0.3s ease;
}
@keyframes expand {
  from {
    width: 100px;
    height: 100px;
  }
  to {
    width: 200px;
    height: 200px;
  }
}
```

### ❌ Deeply nested auto-height containers in hot paths

```javascript
// ❌ Each change to inner content forces all ancestor heights to recalculate
// (all auto-height ancestors must re-layout)
deeplyNested.textContent = newText;
// If every ancestor is height: auto, this propagates up to body
```

---

## 17. Common Mistakes

### Mistake 1 — Misunderstanding the z-index stacking context

```javascript
// ❌ z-index: 9999 doesn't always win
// It depends on the stacking contexts of the competing elements
```

### Mistake 2 — Expecting `height: 100%` to work without explicit parent height

```css
.child {
  height: 100%;
}
/* ❌ Only works if parent has explicit height */
/* If parent is height: auto, 100% has nothing to reference */

/* ✅ Solutions: */
.parent {
  height: 500px;
} /* explicit height */
.parent {
  min-height: 100vh;
} /* viewport relative */
.parent {
  display: flex;
} /* flex container stretches children */
html,
body {
  height: 100%;
} /* propagate from root */
```

### Mistake 3 — Margin collapsing surprises

```css
.section {
  margin-bottom: 40px;
}
.content {
  margin-top: 40px;
}

/* Gap between: 40px (not 80px!) — margins collapsed */
/* Especially confusing when the first child's margin "leaks" out of parent */
```

### Mistake 4 — `transform` breaking `position: fixed` descendants

```javascript
// Adding transform to an ancestor breaks fixed positioning of all descendants
// This is a common bug when adding scroll-linked animations
element.style.transform = `translateY(${scrollY * 0.5}px)`;
// Now any position:fixed children of element are no longer fixed to viewport!
```

---

## 18. Interview-Level Explanation

> **"What is layout/reflow? What triggers it? How do you minimize it?"**

**Strong answer:**

> "Layout — also called reflow — is the browser stage where it computes the exact position and size of every visible element. It takes the styled render tree and produces a layout tree with precise box model geometry: x/y positions, width/height, margins, padding, borders, and where text lines break.
>
> Layout is triggered by any CSS change that affects geometry — width, height, margins, padding, display, position, font-size, flex and grid properties. It's also triggered by DOM insertions and removals. Critically, certain JavaScript reads also force immediate layout: `offsetWidth`, `getBoundingClientRect`, `scrollHeight`, and others. These force the browser to run layout synchronously to return an accurate value.
>
> The most common performance problem is layout thrashing — alternating style writes and geometry reads in a loop. Each read after a write forces a full synchronous layout recalculation. With 200 elements, that's 200 full layouts per frame instead of one. The fix is to batch all reads together before all writes — read all measurements upfront, then apply all changes.
>
> For animation, use `transform` and `opacity` instead of `top`/`left` or `width`/`height`. These properties are handled entirely by the GPU compositor — they don't trigger layout or paint at all.
>
> For architectural containment, CSS `contain: layout` tells the browser that layout effects cannot escape an element. This means updating one widget can't trigger layout for the rest of the page — the browser can limit recalculation to just that widget's subtree. This is underused and very effective for dashboard-style UIs with many independently updating components."

---

## 19. Exercises

### Exercise 1 — Box model calculation

Given this CSS:

```css
.box {
  box-sizing: content-box;
  width: 200px;
  padding: 15px 20px;
  border: 3px solid black;
  margin: 10px;
}
```

Calculate:

1. Content width
2. Total rendered width (including padding and border)
3. Total space occupied on page (including margin)

<details>
<summary>Answer</summary>

```
1. Content width: 200px (box-sizing: content-box — width is content only)

2. Total rendered width (border-box width):
   200px (content) + 20px (padding-left) + 20px (padding-right)
                   + 3px (border-left) + 3px (border-right)
   = 246px

3. Total space on page (margin-box width):
   246px + 10px (margin-left) + 10px (margin-right)
   = 266px

If box-sizing were border-box:
  Content width: 200 - 20 - 20 - 3 - 3 = 154px
  Rendered width: 200px
  Space on page: 220px
```

</details>

---

### Exercise 2 — Find the layout thrashing

```javascript
function updateDashboard(widgets) {
  const containerHeight = dashboard.offsetHeight;

  widgets.forEach((widget) => {
    const currentHeight = widget.offsetHeight;
    const targetHeight = containerHeight / widgets.length;

    widget.style.height = targetHeight + "px";

    if (currentHeight !== targetHeight) {
      const label = widget.querySelector(".label");
      label.style.fontSize = currentHeight > 100 ? "14px" : "12px";

      const labelWidth = label.offsetWidth;
      label.style.maxWidth = targetHeight * 0.8 + "px";
    }
  });
}
```

Identify every forced synchronous layout and provide the batched fix.

<details>
<summary>Solution</summary>

```javascript
// Thrashing points:
// 1. dashboard.offsetHeight            — READ (line 2)
// 2. widget.offsetHeight per widget    — READ inside loop (line 5)
// 3. widget.style.height = ...         — WRITE (line 6), invalidates
// 4. label.offsetWidth                 — READ after WRITE (line 11), forced layout
// 5. label.style.maxWidth = ...        — WRITE again (line 12)
// Pattern: READ → loop(READ → WRITE → READ → WRITE) = thrashing

// FIXED VERSION:
function updateDashboard(widgets) {
  // --- READ PHASE ---
  const containerHeight = dashboard.offsetHeight;

  const measurements = widgets.map((widget) => ({
    widget,
    label: widget.querySelector(".label"),
    currentHeight: widget.offsetHeight,
    labelWidth: widget.querySelector(".label").offsetWidth,
  }));

  // --- COMPUTE PHASE ---
  const targetHeight = containerHeight / widgets.length;

  // --- WRITE PHASE ---
  measurements.forEach(({ widget, label, currentHeight, labelWidth }) => {
    widget.style.height = targetHeight + "px";

    if (currentHeight !== targetHeight) {
      label.style.fontSize = currentHeight > 100 ? "14px" : "12px";
      label.style.maxWidth = targetHeight * 0.8 + "px";
    }
  });
  // One layout total (the initial two reads at the start)
}
```

</details>

---

### Exercise 3 — Stacking context puzzle

```html
<div style="position: relative; z-index: 1;">
  <!-- A -->
  <div style="position: relative; z-index: 100;">
    <!-- B -->
    I am B
  </div>
</div>

<div style="position: relative; z-index: 2;">
  <!-- C -->
  <div style="position: relative; z-index: 1;">
    <!-- D -->
    I am D
  </div>
</div>
```

What is the visual stacking order from back to front?

<details>
<summary>Answer</summary>

```
Stacking order (back to front):
  A (z-index: 1 in root context)
    B (z-index: 100, but WITHIN A's stacking context)
  C (z-index: 2 in root context — above A entirely)
    D (z-index: 1, within C's stacking context)

Visual order from back to front: A → B → C → D

Even though B has z-index: 100 and D has z-index: 1,
D appears ON TOP of B because:
  - C (z-index: 2) > A (z-index: 1) in the root stacking context
  - All of C's children (including D) paint on top of all of A's children (including B)
  - B's z-index: 100 only matters WITHIN A's stacking context — it doesn't compete with D
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](./01-rendering-pipeline.md) — Where layout fits in the full pipeline
- [`browser-internals/03-cssom.md`](./03-cssom.md) — Style computation before layout
- [`browser-internals/05-paint-repaint.md`](./05-paint-repaint.md) — What happens after layout
- [`performance/03-layout-thrashing.md`](../performance/03-layout-thrashing.md) — Fixing layout thrashing in depth
- [`rendering/01-dom-batching.md`](../rendering/01-dom-batching.md) — Batching DOM mutations

---

<div align="center">

**Next:** [`browser-internals/05-paint-repaint.md`](./05-paint-repaint.md) →

</div>
