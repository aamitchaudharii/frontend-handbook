# 06 — Composite Layers

> **"Compositing is where the browser stops being a document renderer and starts being a GPU-accelerated application. Once you understand how layers work, you understand why `transform` is free, why `will-change` creates memory pressure, and why some animations are butter-smooth on a phone while others destroy it."**

Compositing is the final stage of the rendering pipeline — the GPU merges painted layer bitmaps into the final screen image. It's also the stage that, when used correctly, enables 60fps animations that can't be blocked by JavaScript. This document covers how the compositor thread works, how layers are created and managed, what truly runs "on the GPU," and the tradeoffs between compositing performance and memory.

---

## 📚 Table of Contents

1. [What Compositing Is](#1-what-compositing-is)
2. [The Compositor Thread](#2-the-compositor-thread)
3. [How Layers Are Created](#3-how-layers-are-created)
4. [Layer Promotion — Implicit vs Explicit](#4-layer-promotion--implicit-vs-explicit)
5. [What Runs on the Compositor Thread](#5-what-runs-on-the-compositor-thread)
6. [The Layer Tree](#6-the-layer-tree)
7. [Layer Memory Cost](#7-layer-memory-cost)
8. [Layer Squashing](#8-layer-squashing)
9. [Compositing and the 16ms Frame Budget](#9-compositing-and-the-16ms-frame-budget)
10. [will-change — The Right Way](#10-will-change--the-right-way)
11. [transform: translateZ(0) — The Old Hack](#11-transform-translatez0--the-old-hack)
12. [Debugging Layers in DevTools](#12-debugging-layers-in-devtools)
13. [Compositing in Scroll](#13-compositing-in-scroll)
14. [Compositing vs Layout vs Paint — When Each Triggers](#14-compositing-vs-layout-vs-paint--when-each-triggers)
15. [Good Practices](#15-good-practices)
16. [Bad Practices](#16-bad-practices)
17. [Common Mistakes](#17-common-mistakes)
18. [Interview-Level Explanation](#18-interview-level-explanation)
19. [Exercises](#19-exercises)

---

## 1. What Compositing Is

After layout computes geometry and paint fills in pixels, compositing assembles the final frame. Think of it as the browser's equivalent of video editing software compositing layers into a single output:

```
BEFORE COMPOSITING:
  Layer 1 (background):  ████████████████████████████████
  Layer 2 (navigation):  ░░░░░░░░[NAV BAR]░░░░░░░░░░░░░░
  Layer 3 (modal):       ░░░░░░░░░░[   MODAL   ]░░░░░░░░
  Layer 4 (tooltip):     ░░░░░░░░░░░░░░░░░[TIP]░░░░░░░░░

AFTER COMPOSITING (GPU merges them):
  ┌────────────────────────────────────────────────────┐
  │ background content                                  │
  │ ┌──────────────────────────────────────────────┐  │
  │ │ NAV BAR                                       │  │
  │ └──────────────────────────────────────────────┘  │
  │          ┌────────────────┐                        │
  │          │    MODAL       │                        │
  │          │                │  [TIP]                 │
  │          └────────────────┘                        │
  └────────────────────────────────────────────────────┘
         FINAL SCREEN FRAME
```

Compositing is handled by a **separate thread** and by the GPU. This separation is what makes compositor-only animations immune to JavaScript blocking.

---

## 2. The Compositor Thread

The compositor thread is completely separate from the main JavaScript thread. It has its own event loop and communicates with the main thread via messages.

```
MAIN THREAD (JavaScript):
  ├── Parses JavaScript
  ├── Executes event handlers
  ├── Runs garbage collection
  ├── Performs style recalculation
  ├── Performs layout
  ├── Generates display lists (paint instructions)
  └── Sends commit to compositor

COMPOSITOR THREAD (independent):
  ├── Receives layer tree + display lists from main thread
  ├── Sends tiles to raster threads
  ├── Receives rasterized bitmaps from raster threads
  ├── Applies transforms/opacity from animations
  ├── Handles scroll input directly (without main thread)
  └── Submits frames to GPU / display
```

### The Critical Separation

```
Scenario: Main thread is busy (running JavaScript for 200ms)

Main thread: [JS: 200ms long task]

Compositor thread (during those 200ms):
  Frame 1: apply transform animation ✅ smooth
  Frame 2: apply transform animation ✅ smooth
  Frame 3: apply transform animation ✅ smooth
  ...continues at 60fps regardless of main thread
```

This is why `transform` and `opacity` animations can't be blocked by JavaScript — the compositor thread runs them independently. Any animation touching layout or paint must go through the main thread, which CAN be blocked.

---

## 3. How Layers Are Created

Not every DOM element gets its own compositor layer. The browser creates layers strategically to balance compositing benefits against memory cost.

### The Layer Decision

The browser creates a new compositor layer when:

```
1. The element needs to be independently composited
   (to avoid repainting other elements when it changes)

2. The element is GPU-accelerated
   (transforms, opacity animations handled by GPU)

3. The element has content that must be overlaid
   (video, canvas, iframes — always on their own layer)
```

### Layer Creation Rules

```css
/* These create compositor layers: */

/* 1. Elements with will-change set to a compositable property */
will-change: transform;
will-change: opacity;
will-change: filter;

/* 2. Elements with 3D transforms */
transform: translateZ(0);
transform: translate3d(0, 0, 0);
transform: rotateY(45deg);
perspective: 1000px;

/* 3. Elements with CSS animations or transitions on compositor properties */
transition: transform 0.3s;  /* element being animated */

/* 4. Fixed/sticky positioned elements */
position: fixed;
position: sticky;

/* 5. Elements with overflow: scroll that use accelerated scrolling */
overflow: scroll; /* on touch devices */

/* 6. Canvas, video, iframe elements */
<canvas>  <video>  <iframe>  <webgl>

/* 7. Elements with backface-visibility: hidden */
backface-visibility: hidden;

/* 8. Elements overlapping composited layers (sometimes — browser's choice) */
/* If element A is a layer and element B visually overlaps A, the browser
   may promote B to its own layer to avoid incorrect compositing */
```

---

## 4. Layer Promotion — Implicit vs Explicit

There are two ways an element becomes its own compositor layer:

### Explicit Promotion

You explicitly tell the browser to create a layer:

```css
/* Explicit: developer intentionally promotes */
.slider {
  will-change: transform;
  /* Browser: "this will animate — create layer now" */
}

.parallax-element {
  transform: translateZ(0); /* forces layer creation */
}
```

### Implicit Promotion

The browser promotes an element without you asking — to maintain correct visual output:

```
Overlap promotion:
  Layer A (z-order = 1): background layer
  Element B (z-order = 2): overlaps A, not a layer

  → If A's transform changes, B's position relative to A would change
  → Browser must know where B is relative to A to composite correctly
  → Browser promotes B to its own layer automatically

This is called "overlap promotion" and can cause unexpected layer explosion.
```

```html
<!-- Example of implicit overlap promotion cascade -->
<div
  id="animated"
  style="transform: translateZ(0); width: 300px; height: 300px;"
>
  <!-- This element is a compositor layer -->
</div>

<!-- If many elements overlap #animated, browser may promote ALL of them -->
<!-- to their own layers to handle compositing correctly -->
<!-- This can create hundreds of unexpected layers -->
```

---

## 5. What Runs on the Compositor Thread

Only specific operations can run on the compositor thread without involving the main thread:

### Compositor-Thread Operations

```
✅ Runs entirely on compositor thread (never touches main thread):
  transform: translate() — moves layer position
  transform: scale()     — scales layer bitmap
  transform: rotate()    — rotates layer bitmap
  opacity               — adjusts layer transparency
  scroll                — moves content (GPU handles scroll position)
  filter: brightness()   — GPU filter pass
  filter: contrast()     — GPU filter pass
  filter: saturate()     — GPU filter pass
```

### Main-Thread Operations (Cannot Be on Compositor)

```
❌ Must go through main thread (can be blocked by JavaScript):
  background-color       — requires repaint
  color                  — requires repaint
  box-shadow             — requires repaint
  width/height           — requires layout + repaint
  position/top/left      — requires layout + repaint
  filter: blur()         — CPU rasterized (often)
  clip-path (complex)    — requires repaint
```

### How the Compositor Handles Transform Animations

```
Main thread (sets up animation):
  1. Creates animation timeline with keyframes
  2. Records: "element X should animate transform from A to B over T ms"
  3. Sends commit to compositor thread

Compositor thread (per frame, forever):
  1. Check timeline: current position at time t
  2. Interpolate transform value (easing function applied on GPU)
  3. Apply transform matrix to layer's GPU texture
  4. Composite with other layers
  5. Submit frame to display

Main thread: free to do anything else
JavaScript: can run, GC can run — animation unaffected
```

---

## 6. The Layer Tree

The browser maintains a **layer tree** — a hierarchical structure of compositor layers that mirrors the stacking context hierarchy.

```
Layer Tree Structure:

DocumentLayer (root)
  ├── BackgroundLayer (background-color of body)
  │
  ├── ContentLayer (normal flow content — one layer for most of the page)
  │   ├── [text nodes]
  │   ├── [inline images]
  │   └── [block elements without own layers]
  │
  ├── NavigationLayer (position: fixed nav — own layer)
  │
  ├── AnimatedCardLayer (will-change: transform — own layer)
  │
  └── VideoLayer (video element — always own layer)
```

### Layer Tree Commit

The main thread sends "commits" to the compositor thread when something changes:

```
Main thread commits:
  1. Updated layer tree structure (elements added/removed from layers)
  2. Updated display lists (new paint commands for repainted layers)
  3. Updated animation data (new keyframes, timing changes)
  4. Updated scroll offsets

Compositor thread receives commit and:
  1. Updates its copy of the layer tree
  2. Sends dirty tiles to raster threads
  3. Continues animating with new data
```

---

## 7. Layer Memory Cost

Every compositor layer consumes **GPU memory** proportional to its pixel dimensions.

### Memory Formula

```
GPU memory per layer = width × height × bytes_per_pixel

For RGBA (4 bytes per pixel):
  100 × 100 pixels  = 40,000 bytes   ≈ 40KB
  800 × 600 pixels  = 1,920,000 bytes ≈ 1.9MB
  1920 × 1080 pixels = 8,294,400 bytes ≈ 8.3MB
  Full 1440p page    = potentially 100MB+
```

### Device Memory Limits

```
Desktop (dedicated GPU, 4-8GB VRAM):
  → GPU memory for layers: generous, rarely a problem

Mobile (shared RAM/VRAM, 2-6GB total):
  → Memory for GPU textures: 200-500MB typical budget
  → Too many layers: browser starts evicting textures
  → Evicted textures must be re-rasterized when needed
  → Causes: checkerboard flicker, scroll jank, animation stutters

Low-end mobile (1-2GB RAM):
  → Critical to minimize layer count
  → Even 10 unnecessary large layers can cause problems
```

### Practical Memory Calculation

```javascript
// Calculate memory impact of promoting elements to layers
function estimateLayerMemory(selector) {
  const elements = document.querySelectorAll(selector);
  let totalBytes = 0;

  elements.forEach((el) => {
    const rect = el.getBoundingClientRect();
    const bytes = rect.width * rect.height * 4; // 4 bytes per RGBA pixel
    totalBytes += bytes;
  });

  return {
    elementCount: elements.length,
    totalMB: (totalBytes / 1024 / 1024).toFixed(2),
    perElementMB: (totalBytes / elements.length / 1024 / 1024).toFixed(2),
  };
}

// Before applying will-change to .list-item:
estimateLayerMemory(".list-item");
// { elementCount: 1000, totalMB: "468.75", perElementMB: "0.47" }
// → 469MB GPU memory for 1000 list items — too much!
```

---

## 8. Layer Squashing

The browser doesn't necessarily create one layer per promoted element. It uses **layer squashing** to merge nearby layers with compatible properties into a single layer.

### When Squashing Happens

```
Elements A, B, C all have will-change: transform
  → Could be 3 separate layers (3× memory)

If they don't overlap and have compatible blend modes:
  → Browser may squash them into 1 layer (1× memory)
  → Still gets compositing benefit for position changes
  → Less memory pressure
```

### When Squashing Doesn't Happen

```
Elements are squashed into SEPARATE layers when:
  - They overlap each other
  - They have different blend modes
  - Squashing would create incorrect visual output
  - One has a CSS filter the other doesn't
```

### Checking Layer Counts

```javascript
// Chrome DevTools: check layer count in Layers panel
// DevTools → Layers tab → see all layers with memory estimates

// Or via Performance:
// Record → scroll/interact → Stop
// In Layers timeline: see peak layer count
```

---

## 9. Compositing and the 16ms Frame Budget

How compositing fits into the frame budget:

```
FRAME BUDGET (16.67ms at 60fps):

Main thread:
  JavaScript execution:    ~4ms
  Style recalculation:     ~0.5ms
  Layout:                  ~2ms
  Update layer tree:       ~0.5ms
  ─────────────────────────────
  Total main thread:       ~7ms

Raster threads (parallel):
  Rasterize dirty tiles:   ~2ms (parallel, doesn't block main thread)

Compositor thread:
  Apply animations:        ~0.5ms
  Submit frame to GPU:     ~0.5ms
  ─────────────────────────────────
  Total compositor:        ~1ms

GPU:
  Composite layers:        ~2ms
  Display flip:            ~1ms
  ─────────────────────────────────
  Total GPU:               ~3ms

TOTAL FRAME:               ~13ms (leaves ~3.5ms margin)
```

### Compositor-Only Frame (Transform/Opacity Animation)

When only compositor-only properties change:

```
Compositor-only frame (no JS, no layout, no paint):

Main thread:     [idle — no work needed]

Compositor thread:
  Apply animation update:  ~0.1ms
  Submit to GPU:           ~0.5ms

GPU:
  Composite layers:        ~1ms
  Display:                 ~0.5ms

TOTAL: ~2ms
Even fits in a 120fps frame (8.33ms)!
```

---

## 10. will-change — The Right Way

`will-change` is a hint to the browser to prepare a GPU layer for an element that will be animated. Used correctly, it eliminates promotion overhead at animation start. Used incorrectly, it wastes memory.

### When `will-change` Is Appropriate

```css
/* ✅ Elements that are frequently and knowingly animated */
.notification-badge {
  will-change: transform; /* always bounces in — appropriate to pre-promote */
}

.loading-spinner {
  will-change: transform; /* continuously rotating — always needs layer */
}

.parallax-layer {
  will-change: transform; /* scroll-linked transform — constantly animating */
}
```

### Dynamic `will-change` (Better for Infrequent Animations)

```javascript
// ✅ Add will-change before animation, remove after
const card = document.querySelector(".card");

card.addEventListener("mouseenter", () => {
  // Promote just before animation starts
  card.style.willChange = "transform";
});

card.addEventListener("transitionend", () => {
  // Demote after animation ends — release GPU memory
  card.style.willChange = "auto";
});

// Or with class:
card.addEventListener("mouseenter", () => card.classList.add("animating"));
card.addEventListener("mouseleave", () => {
  card.addEventListener(
    "transitionend",
    () => card.classList.remove("animating"),
    { once: true },
  );
});
```

```css
.card.animating {
  will-change: transform;
}
```

### Properties `will-change` Accepts

```css
will-change: auto; /* no hint — default */
will-change: scroll-position; /* hint: will scroll */
will-change: contents; /* hint: children will change */
will-change: transform; /* hint: will transform */
will-change: opacity; /* hint: will change opacity */
will-change: transform, opacity; /* multiple hints */

/* Avoid: */
will-change: all; /* too broad — promotes for everything */
```

---

## 11. transform: translateZ(0) — The Old Hack

Before `will-change` was widely supported, developers used `transform: translateZ(0)` (and `translate3d(0,0,0)`) to force compositor layer creation. This is a hack that still works but has important caveats.

### How It Works

```css
.element {
  transform: translateZ(0);
  /* translateZ(0) = move 0px on the Z axis
     Result: visually unchanged
     Side effect: forces a 3D rendering context → compositor layer */
}
```

### Why It Was Used

```
Old browsers (pre-will-change):
  - No explicit way to create compositor layers
  - Any 3D transform forced a layer
  - translateZ(0) became the standard hack

Modern browsers (post-will-change):
  - will-change: transform is the correct way
  - translateZ(0) still works but is more opaque in intent
  - will-change: transform is the semantic replacement
```

### Difference from `will-change`

```css
/* will-change: signals intent — browser may or may not create layer */
.element {
  will-change: transform;
}
/* Browser: "probably create a layer, but I can decide" */

/* transform: translateZ(0): forces layer creation unconditionally */
.element {
  transform: translateZ(0);
}
/* Browser: "3D transform present — must create layer" */
```

In practice, both reliably create layers in modern browsers.

---

## 12. Debugging Layers in DevTools

### The Layers Panel

```
DevTools → More Tools → Layers

Shows:
  - 3D view of all compositor layers (tilt/rotate to see depth)
  - Each layer's dimensions and memory usage
  - Why the layer was created (mouseover → "Compositing reasons")
  - Total layer memory (bottom of panel)

Compositing reason examples:
  "Has a will-change attribute"
  "Element has 3D CSS property"
  "Is the root element"
  "Element has overflow clip that is not handled by the root scroll layer"
  "Has a position:fixed descendant"
  "Has a CSS animation affecting transform or opacity"
```

### Layer Borders (Quick Visual Check)

```
DevTools → Rendering → Layer Borders

Orange outline: compositor layer boundary
Blue dotted: tile boundary within a layer

What to look for:
  ✅ Few orange boxes — minimal layers
  ❌ Many orange boxes covering entire page — layer explosion
  ❌ Large orange boxes for rarely-animated elements — unnecessary layers
```

### Identifying Unexpected Layers

```javascript
// Programmatic layer detection (approximate)
// Elements with will-change = potential layers
document.querySelectorAll("*").forEach((el) => {
  const style = getComputedStyle(el);
  if (style.willChange !== "auto") {
    console.log("Layer candidate:", el, style.willChange);
  }
  if (style.transform !== "none" && style.transform.includes("matrix3d")) {
    console.log("3D transform layer:", el);
  }
});
```

---

## 13. Compositing in Scroll

Scrolling is a primary use case for compositing. Modern browsers handle scrolling on the compositor thread for smooth performance.

### Compositor Thread Scrolling

```
User swipes/scrolls:

Without compositor scrolling:
  Input event → Main thread → layout recalculation → repaint → display
  Any main thread work delays scroll response

With compositor thread scrolling:
  Input event → Compositor thread moves content layer → display
  Main thread not involved for scroll itself
```

### When Scroll Leaves the Compositor Thread

Scroll is handled on the compositor thread UNLESS:

```javascript
// ❌ Passive: false — forces main thread involvement
window.addEventListener("scroll", handler, { passive: false });
// Browser cannot optimize scroll — must check if handler calls preventDefault()

// ✅ Passive: true (default for touch events in modern browsers)
window.addEventListener("scroll", handler, { passive: true });
// Browser: "handler won't prevent scroll — I can scroll on compositor thread"
// Scroll remains smooth even if handler is slow
```

### Sticky Positioning and Compositing

```css
.sticky-header {
  position: sticky;
  top: 0;
}
/* Sticky elements must be composited separately from the scroll layer
   Browser creates a compositor layer for sticky elements
   This enables the "stick" behavior without main thread involvement */
```

### Scroll-Linked Animations and Compositing

```javascript
// ❌ Scroll handler modifying styles — goes through main thread
window.addEventListener('scroll', () => {
  const ratio = scrollY / maxScroll;
  element.style.opacity = 1 - ratio; // forces style recalculation on scroll
});

// ✅ CSS scroll-linked animations (compositor thread)
@keyframes fade-on-scroll {
  from { opacity: 1; }
  to   { opacity: 0; }
}

.element {
  animation: fade-on-scroll linear;
  animation-timeline: scroll(); /* CSS scroll-driven animation */
  /* Compositor thread handles this — no JavaScript */
}
```

---

## 14. Compositing vs Layout vs Paint — When Each Triggers

The definitive reference for the rendering pipeline cost of common operations:

```
OPERATION                        LAYOUT?  PAINT?  COMPOSITE?
─────────────────────────────────────────────────────────────
element added to DOM             ✅       ✅       ✅
element removed from DOM         ✅       ✅       ✅
element.style.width changed      ✅       ✅       ✅
element.style.height changed     ✅       ✅       ✅
element.style.display changed    ✅       ✅       ✅
element.style.margin changed     ✅       ✅       ✅
element.style.padding changed    ✅       ✅       ✅
element.style.position changed   ✅       ✅       ✅
element.style.top/left changed   ✅       ✅       ✅
element.style.font-size changed  ✅       ✅       ✅
element.classList.add (geometry) ✅       ✅       ✅
element.style.color changed      ❌       ✅       ✅
element.style.background changed ❌       ✅       ✅
element.style.box-shadow changed ❌       ✅       ✅
element.style.border-radius      ❌       ✅       ✅
element.style.visibility changed ❌       ✅       ✅
element.style.filter: blur()     ❌       ✅       ✅
element.style.filter: brightness ❌       ❌       ✅
element.style.transform changed  ❌       ❌       ✅  ⭐
element.style.opacity changed    ❌       ❌       ✅  ⭐
scroll event fires               ❌       ❌       ✅
```

---

## 15. Good Practices

### ✅ Promote only what you know will animate

```css
/* ✅ Promoting an element that genuinely needs it */
.animated-menu {
  /* This element slides in from the left on every page load */
  will-change: transform; /* pre-promote — justified */
}

/* ✅ For infrequent animations: dynamic promotion */
.expandable-card.expanding {
  will-change: transform, opacity;
}
```

### ✅ Remove `will-change` after animation completes

```javascript
// ✅ Clean up after animation — release GPU memory
element.addEventListener(
  "transitionend",
  () => {
    element.style.willChange = "auto";
  },
  { once: true },
);
```

### ✅ Verify layer counts stay reasonable

```
Rule of thumb: < 30 compositor layers on any page
If you see 100+ layers: investigate the cause

DevTools → Layers panel → check "Total layers" and memory
```

### ✅ Use passive event listeners for scroll

```javascript
// ✅ Allows compositor thread scroll optimization
window.addEventListener("scroll", onScroll, { passive: true });
element.addEventListener("touchmove", onTouchMove, { passive: true });
```

### ✅ Use CSS animations for compositor-only properties (not JavaScript)

```css
/* ✅ CSS animation: compositor can handle entirely */
.spinner {
  animation: rotate 1s linear infinite;
  will-change: transform;
}
@keyframes rotate {
  to { transform: rotate(360deg); }
}

/* vs. JavaScript (requires main thread involvement per frame): */
function animate() {
  angle += 6;
  spinner.style.transform = `rotate(${angle}deg)`;
  requestAnimationFrame(animate);
}
```

---

## 16. Bad Practices

### ❌ Applying `will-change` to everything

```css
/* ❌ Layer explosion — browsers have been known to crash */
* {
  will-change: transform;
}
.container * {
  will-change: opacity;
}
.list li {
  will-change: transform;
} /* 1000 items = 1000 GPU textures */
```

### ❌ Never removing `will-change`

```css
/* ❌ will-change applied permanently to rarely-animated elements */
.tooltip {
  will-change: transform; /* shows 2× per day — GPU memory wasted 24/7 */
}

/* ✅ Apply dynamically just before animation */
```

### ❌ Causing implicit layer explosion with fixed elements

```html
<!-- ❌ A fixed element + overlapping content = many implicit layers -->
<div style="position: fixed; z-index: 100;">Fixed header</div>
<!-- Every element that visually overlaps the fixed header
     may be promoted to its own layer by the browser
     If hundreds of elements overlap: hundreds of layers -->

<!-- ✅ Keep fixed elements away from scrolling content visually,
       or contain the content layers -->
```

### ❌ Using `transform: translateZ(0)` as a "fix everything" hack

```css
/* ❌ Cargo-culting translateZ(0) — creates layers everywhere */
.slow-animation {
  transform: translateZ(0);
}
.flickering-text {
  transform: translateZ(0);
}
.dropdown-menu {
  transform: translateZ(0);
}

/* Each creates a compositor layer — may not actually help */
/* May create the overlap promotion cascade problem */
```

---

## 17. Common Mistakes

### Mistake 1 — Expecting `will-change` to speed up paint-triggering animations

```css
/* ❌ Misconception: will-change makes background-color animation fast */
.card {
  will-change: background-color; /* NOT a compositable property */
  transition: background-color 0.3s;
}
/* background-color still requires paint every frame
   will-change has no special effect here */

/* ✅ background-color transitions always involve paint
      Accept it, or restructure to use opacity overlay */
```

### Mistake 2 — z-index interactions with compositor layers

```css
/* GOTCHA: composited elements create new stacking contexts */
/* This can break z-index ordering with non-composited siblings */

.above {
  position: relative;
  z-index: 100;
}
.composited {
  will-change: transform;
  z-index: 1;
}

/* Even though .above has higher z-index,
   .composited's layer may render above it
   depending on compositing order */
```

### Mistake 3 — Forgetting that `transform` doesn't affect layout

```javascript
// transform moves the element visually but NOT in the layout tree
element.style.transform = "translateX(100px)";

// element.getBoundingClientRect() reflects the VISUAL position (with transform)
// element.offsetLeft reflects the LAYOUT position (without transform)

element.getBoundingClientRect().left; // 100px offset
element.offsetLeft; // original position (no transform)
```

### Mistake 4 — Large layer with only a small animated portion

```css
/* ❌ Will-change on a very large element when only a small part animates */
.full-page-container {
  will-change: transform; /* entire page is one giant GPU texture */
}

/* ✅ Apply will-change only to the small animated child */
.animated-child {
  will-change: transform;
}
```

---

## 18. Interview-Level Explanation

> **"What is compositing? What makes `transform` and `opacity` different from other CSS properties?"**

**Strong answer:**

> "Compositing is the final rendering stage where the GPU merges multiple layer bitmaps into the final screen frame. Each compositor layer — a separate GPU texture — is assembled together using the GPU's blending capabilities.
>
> What makes `transform` and `opacity` special is that they run on the **compositor thread** — completely separate from the main JavaScript thread. When an animation only uses these properties, the compositor thread reads the animation timeline, applies the transform matrix or opacity value to the GPU texture, and submits the frame — all without touching the main thread. This means the animation cannot be blocked by JavaScript execution, garbage collection, or layout work happening on the main thread.
>
> Any other CSS property that triggers paint — like `background-color`, `box-shadow`, or `width` — requires the main thread to re-rasterize pixels into the layer bitmap. This work competes with JavaScript and can be blocked by long tasks.
>
> Compositor layers are created when you use `will-change: transform`, `transform: translateZ(0)`, `position: fixed`, or certain CSS animations. Each layer consumes GPU memory proportional to its pixel area — on mobile, this can be significant. A 1920×1080 layer uses about 8MB of GPU memory, so creating many layers unnecessarily causes memory pressure.
>
> The most common misuse is applying `will-change: transform` to every element 'just in case.' This wastes GPU memory constantly and can cause implicit layer promotion cascades — where the browser promotes overlapping elements to their own layers, leading to hundreds of unintended layers. The correct pattern is to apply `will-change` dynamically just before an animation starts, and remove it afterward with `will-change: auto`."

---

## 19. Exercises

### Exercise 1 — Layer count estimation

Given this page structure, predict how many compositor layers will be created and explain why:

```html
<body>
  <header style="position: fixed; top: 0;">Navigation</header>
  <main>
    <video src="hero.mp4" autoplay muted></video>
    <section class="parallax" style="transform: translateZ(0);">
      Content
    </section>
    <ul>
      <!-- 50 list items, no special styles -->
      <li>Item 1</li>
      ...
      <li>Item 50</li>
    </ul>
    <div class="spinner" style="will-change: transform;">Loading</div>
  </main>
</body>
```

<details>
<summary>Answer</summary>

```
Guaranteed compositor layers:

1. Root layer (DocumentLayer) — always exists
2. header (position: fixed) — fixed elements always get own layer
3. video — media elements always get own layer
4. .parallax (transform: translateZ(0)) — 3D transform forces layer
5. .spinner (will-change: transform) — explicit will-change

Possible additional layers:
6. Overlap promotion: elements visually overlapping the fixed header
   (the parallax section, the list, etc.) MAY get promoted
   depending on browser heuristics — could be several more

Total: minimum 5 layers, possibly 10-15 depending on overlap promotion

The 50 list items: NO own layers (no compositing hints, no 3D transforms)
They share the root content layer.

Memory estimate (rough):
  Root content layer (1920×3000px): ~23MB
  Header (1920×60px): ~0.5MB
  Video layer: ~8MB (1920×1080 decoded frame)
  Parallax (1920×600px): ~4.6MB
  Spinner (50×50px): ~0.01MB
  Total: ~36MB GPU memory (acceptable)
```

</details>

---

### Exercise 2 — Fix the layer explosion

This code is causing 500+ compositor layers on mobile:

```javascript
// Item list with hover animations
const items = document.querySelectorAll(".feed-item");

items.forEach((item) => {
  item.style.willChange = "transform, opacity";

  item.addEventListener("mouseenter", () => {
    item.style.transform = "scale(1.02)";
    item.style.opacity = "0.95";
  });

  item.addEventListener("mouseleave", () => {
    item.style.transform = "";
    item.style.opacity = "";
  });
});
```

Rewrite to use layers efficiently.

<details>
<summary>Solution</summary>

```javascript
// ✅ Only promote the item being hovered, demote after
const items = document.querySelectorAll(".feed-item");

items.forEach((item) => {
  // NO will-change applied initially
  // 500 items × will-change = 500 GPU textures = bad

  item.addEventListener("mouseenter", () => {
    // Promote just for THIS item, just for THIS hover
    item.style.willChange = "transform, opacity";

    // Use requestAnimationFrame to ensure layer is created before animation
    requestAnimationFrame(() => {
      item.style.transform = "scale(1.02)";
      item.style.opacity = "0.95";
    });
  });

  item.addEventListener("mouseleave", () => {
    item.style.transform = "";
    item.style.opacity = "";
  });

  item.addEventListener("transitionend", () => {
    // Demote after transition completes — release GPU memory
    item.style.willChange = "auto";
  });
});

// CSS to accompany this:
// .feed-item { transition: transform 0.2s ease, opacity 0.2s ease; }
```

**Result:**

- Initial: 0 extra layers (only the root layer)
- During hover: 1 extra layer (only the hovered item)
- After hover: 0 extra layers (demoted)
- GPU memory: ~0.5MB max (instead of ~240MB for 500 items)

</details>

---

### Exercise 3 — Analyze a performance trace

Using Chrome DevTools:

1. Open a page with a CSS animation that uses `background-color` transitions
2. Open DevTools → Performance → Enable "Screenshots"
3. Record for 3 seconds while hovering the animated element
4. Stop recording
5. In the flame graph, find the animation frames

Answer these questions from the trace:

- Are you seeing "Paint" events during the animation?
- How long does each paint take?
- Is the animation running on the main thread or compositor thread?
- What would need to change to move it to the compositor thread?

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](./01-rendering-pipeline.md) — Full pipeline where compositing fits
- [`browser-internals/05-paint-repaint.md`](./05-paint-repaint.md) — Paint stage before compositing
- [`browser-internals/07-gpu-acceleration.md`](./07-gpu-acceleration.md) — GPU acceleration in detail
- [`animations/03-compositor-animations.md`](../animations/03-compositor-animations.md) — Building compositor-only animations
- [`performance/04-raf-optimization.md`](../performance/04-raf-optimization.md) — rAF and compositor interaction

---

<div align="center">

**Next:** [`browser-internals/07-gpu-acceleration.md`](./07-gpu-acceleration.md) →

</div>
