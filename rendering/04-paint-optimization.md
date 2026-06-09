# 04 — Paint Optimization

> **"Paint is expensive not because the pixels are hard to compute — GPUs are fast at that — but because triggering a repaint means the browser must go back to the CPU, figure out what changed, rasterize it, and upload it to the GPU again. The goal is to make the GPU do more, and the CPU do less."**

Paint optimization is about minimizing the frequency, area, and cost of rasterization — the process of converting display commands into pixel bitmaps. Every repaint consumes CPU time (or GPU time if GPU rasterization is active), requires re-uploading bitmaps to GPU memory, and costs precious milliseconds in the rendering pipeline. This document covers what triggers paint, how to minimize repaints, paint profiling, and the specific CSS properties and patterns that keep repaints cheap.

---

## 📚 Table of Contents

1. [What Painting Is](#1-what-painting-is)
2. [What Triggers Repaint](#2-what-triggers-repaint)
3. [Paint Profiling in DevTools](#3-paint-profiling-in-devtools)
4. [Reducing Paint Area — Dirty Rectangles](#4-reducing-paint-area--dirty-rectangles)
5. [Compositor Layers as Paint Isolation](#5-compositor-layers-as-paint-isolation)
6. [Expensive CSS Properties](#6-expensive-css-properties)
7. [Box Shadow Optimization](#7-box-shadow-optimization)
8. [Border Radius and Clip Paths](#8-border-radius-and-clip-paths)
9. [Background Optimization](#9-background-optimization)
10. [Text Rendering Optimization](#10-text-rendering-optimization)
11. [Animations Without Repaint](#11-animations-without-repaint)
12. [Contain: paint — Paint Isolation](#12-contain-paint--paint-isolation)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. What Painting Is

Paint (rasterization) converts the browser's display list — a series of drawing commands — into pixel bitmaps in memory.

```
THE PAINT PROCESS:

Display list (drawing commands):
  "Fill rect (0,0,100,100) with #4fc3f7"
  "Draw text 'Hello' at (10,15) in 16px sans-serif"
  "Draw circle at (50,50) radius 20 with border #007bff"
  "Draw box-shadow (4px 4px 8px rgba(0,0,0,0.3)) on rect (0,0,100,100)"

Rasterization (CPU or GPU):
  Execute each drawing command → produce pixel bitmap
  Each pixel: compute RGBA value based on all commands affecting it

Result: bitmap in RAM (or VRAM if GPU rasterization)
Then: upload to GPU as texture
Then: compositor assembles textures → final frame

COST FACTORS:
  Area: larger = more pixels to compute
  Complexity: shadows, gradients, blurs = more computation per pixel
  Frequency: every frame = 60× per second
  Layer count: each compositor layer is a separate bitmap
```

---

## 2. What Triggers Repaint

Not all CSS changes trigger layout. Some trigger paint only. Some trigger neither (compositor only):

```
TRIGGERS LAYOUT + PAINT + COMPOSITE (most expensive):
  width, height, top, left, right, bottom
  margin, padding, border
  display, position, float
  font-size, font-family
  overflow

TRIGGERS PAINT + COMPOSITE (no layout):
  color, background-color, background-image
  border-color, border-radius (partially)
  outline, outline-color
  box-shadow
  text-decoration
  visibility

TRIGGERS COMPOSITE ONLY (cheapest):
  opacity
  transform (translate, scale, rotate, skew)
  filter: brightness(), contrast(), saturate(), hue-rotate()
  backdrop-filter
  clip-path (in some cases — depends on GPU support)
```

### Why Some Properties Are Compositor-Only

```
Transform/opacity operate on the GPU texture AFTER rasterization:
  1. Element is rasterized into a GPU texture (once)
  2. Each frame: GPU applies the transform matrix or opacity
     to the existing texture
  3. GPU composites all textures → final frame

No CPU rasterization per frame — GPU matrix math is essentially free.

Color/background-color require rasterization per change:
  1. New color means the pixels in the bitmap are different
  2. CPU (or GPU rasterizer) must re-draw the element
  3. New bitmap uploaded to GPU texture memory
  4. GPU composites textures

This is why transform/opacity animations are smooth while
color animations are expensive.
```

---

## 3. Paint Profiling in DevTools

### Enabling Paint Flashing

```
Chrome DevTools → Rendering tab (More tools → Rendering)
  ✓ Paint Flashing

Green overlay = areas being repainted this frame
Ideal: minimal green on static pages
Expected: green on animated elements
Problem: green flashing on static content during animations
         (indicates paint is being triggered when it shouldn't be)
```

### Performance Panel — Paint Analysis

```
DevTools → Performance → Record

In the flame chart, look for:
  Purple "Paint" events → rasterization happening
  Yellow "Commit" events → uploading bitmaps to GPU
  "Compositing" events → GPU combining layers

Key questions:
  How many Paint events per frame? (ideal: 0 or 1)
  How long does each Paint take? (>2ms is expensive)
  What changed to cause the Paint? (hover effect? animation? scroll?)
  Which layers are being painted? (Layers panel)
```

### Layers Panel

```
DevTools → More tools → Layers

Shows:
  3D view of all compositor layers
  Size and memory usage of each layer
  "Why this layer was created"
  Layer compositing reasons

Use to verify:
  Animated elements have their own layers (no contaminating repaints)
  Static elements share a layer (not promoted unnecessarily)
  Total layer memory is reasonable (< 50MB for typical pages)
```

---

## 4. Reducing Paint Area — Dirty Rectangles

The browser tracks which areas of the screen need repainting (dirty rectangles). Minimize the dirty area to minimize paint cost.

### Spatial Isolation

```css
/* ❌ Changing an element that overlaps many other elements
   forces repainting everything it overlaps */

/* If .tooltip appears over a complex background: */
.tooltip {
  background-color: rgba(0, 0, 0, 0.8);
  /* When tooltip appears: entire overlap area must repaint */
}

/* ✅ Promote animated/toggling elements to their own layer
   so their appearance doesn't contaminate the content below */
.tooltip {
  background-color: rgba(0, 0, 0, 0.8);
  will-change: opacity; /* promotes to layer — composites independently */
}
/* Now tooltip fading in: compositor operation, no repaint of underlying content */
```

### Reduce Element Complexity in High-Frequency Zones

```css
/* ❌ Complex element that gets repainted frequently during scroll */
.sticky-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.6);
  border-bottom: 2px solid rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  /* All of these are expensive to repaint */
}

/* ✅ Simplify or split into layers */
.sticky-header {
  background: #667eea; /* solid color: fast to repaint */
  /* Complex visual effects via ::before/::after as separate layers */
}
.sticky-header::after {
  content: "";
  position: absolute;
  inset: 0;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  pointer-events: none;
  /* This layer only repaints when header itself changes — not on scroll */
}
```

---

## 5. Compositor Layers as Paint Isolation

Compositor layers separate elements so their repaints are independent:

```css
/* The Key Insight:
   When Element A and Element B are on the same layer,
   repainting A requires repainting B (and everything else on that layer).

   When A and B are on separate layers,
   repainting A only repaints A's layer bitmap.
   B's layer bitmap is unchanged. */
```

```javascript
// Scenario: Animating a modal overlay while background content is interactive

/* ❌ Modal on same layer as content:
   Each opacity change → repaint modal + content behind it */
.modal {
  /* no compositor layer promotion */
}

/* ✅ Modal on its own layer:
   Opacity changes: compositor operation only, no repaint */
.modal {
  will-change: opacity, transform;
  /* or: transform: translateZ(0) */
}

/* Each opacity animation frame:
   Before: CPU rasterizes modal + everything behind it
   After:  GPU adjusts opacity on existing modal texture
   Cost reduction: ~10ms → ~0.1ms per frame */
```

### Strategic Layer Creation

```css
/* Promote elements that change frequently and independently */

/* Loading spinner: always animating */
.spinner {
  will-change: transform; /* permanent layer — justified */
}

/* Toast notification: appears/disappears frequently */
.toast {
  will-change: transform, opacity; /* promotes during animation */
}

/* Navigation: position: fixed is auto-promoted */
.nav {
  position: fixed;
} /* browser creates layer automatically */

/* ❌ DON'T promote everything */
.list-item {
  will-change: transform;
} /* 1000 items = 1000 layers = bad */
```

---

## 6. Expensive CSS Properties

### Paint Cost Hierarchy

```
VERY EXPENSIVE (avoid in animated/high-frequency elements):
  filter: blur()              — Gaussian blur scales with radius²
  box-shadow (large spread)   — depends on size and blur radius
  backdrop-filter             — samples and blurs content behind element
  text-stroke (webkit)        — complex rasterization
  SVG filters                 — depends on filter type and area

EXPENSIVE (acceptable for static, avoid animating):
  border-radius               — antialiased corners
  gradients (complex)         — multi-stop, radial
  background-blend-mode       — pixel blending
  outline with large radius   — similar to border-radius

MODERATE:
  box-shadow (small, sharp)   — quick if simple
  gradients (simple)          — 2-stop linear: fast
  border (various styles)     — dotted/dashed slightly slower than solid

CHEAP:
  background-color (solid)    — single fill operation
  border (solid, single-color) — simple path drawing
  color                       — text fill change
```

---

## 7. Box Shadow Optimization

Box shadows are one of the most common paint performance culprits:

```css
/* The cost formula for box-shadow:
   Cost ≈ element_area × blur_radius × (number_of_shadows)² */

/* ❌ Very expensive: large blur on large element */
.hero-section {
  width: 100vw;
  height: 100vh;
  box-shadow: 0 10px 100px rgba(0, 0, 0, 0.5); /* 100px blur on fullscreen = massive */
}

/* ❌ Multiple shadows: compounds the cost */
.card {
  box-shadow:
    0 2px 4px rgba(0, 0, 0, 0.1),
    0 4px 8px rgba(0, 0, 0, 0.1),
    0 8px 16px rgba(0, 0, 0, 0.1),
    0 16px 32px rgba(0, 0, 0, 0.1); /* 4 shadows = 4× paint cost */
}
```

```css
/* ✅ Optimization 1: Use a pseudo-element for the shadow
   — pre-rendered once, promoted to its own layer */
.card {
  position: relative;
}
.card::after {
  content: "";
  position: absolute;
  inset: -5px;
  border-radius: inherit;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.2);
  opacity: 0; /* GPU opacity on separate layer */
  transition: opacity 0.3s;
  will-change: opacity; /* promote layer for transition */
  pointer-events: none;
  z-index: -1;
}
.card:hover::after {
  opacity: 1; /* GPU compositing only — no repaint */
}
```

```css
/* ✅ Optimization 2: filter: drop-shadow instead of box-shadow
   (GPU-accelerated in modern browsers) */
.icon {
  filter: drop-shadow(0 4px 8px rgba(0, 0, 0, 0.2));
  /* drop-shadow respects transparency + is GPU composited */
}
```

```javascript
// ✅ Optimization 3: Avoid animating box-shadow
// Instead: animate opacity of a pre-drawn shadow layer

// ❌ This repaints every frame
element.style.boxShadow = `0 ${shadowSize}px ${blur}px rgba(0,0,0,0.5)`;

// ✅ This is compositor-only
shadowElement.style.opacity = String(shadowOpacity);
```

---

## 8. Border Radius and Clip Paths

```css
/* border-radius has paint cost — especially during animation */

/* ❌ Animating border-radius triggers repaint every frame */
.avatar {
  transition: border-radius 0.3s;
  border-radius: 50%;
}
.avatar:hover {
  border-radius: 4px; /* every frame: rasterize new shape */
}

/* ✅ Animate transform instead — compositor only */
.avatar {
  border-radius: 4px;
  /* Pre-render both states, animate between them via clipPath or mask */
}

/* ✅ For shape morphing: use clip-path with transform */
.shape {
  clip-path: circle(50%); /* static clip — rasterized once */
  transition: transform 0.3s; /* transform = compositor only */
}
```

```css
/* clip-path performance:
   Simple shapes (circle, inset): GPU-accelerated in modern browsers
   Complex polygons: may require CPU rasterization
   Animated clip-path: ALWAYS triggers repaint — expensive */

/* ❌ Animating clip-path: expensive */
.reveal {
  clip-path: polygon(0 0, 0 0, 0 100%, 0 100%);
  transition: clip-path 0.5s;
}
.reveal:hover {
  clip-path: polygon(0 0, 100% 0, 100% 100%, 0 100%);
}

/* ✅ Animate transform + overflow:hidden instead */
.reveal-container {
  overflow: hidden;
}
.reveal {
  transform: translateX(-100%);
  transition: transform 0.5s;
}
.reveal-container:hover .reveal {
  transform: translateX(0);
}
```

---

## 9. Background Optimization

```css
/* PERFORMANCE ORDER (fast → slow): */

/* 1. Fastest: solid color */
background-color: #4fc3f7;

/* 2. Fast: simple 2-stop linear gradient */
background: linear-gradient(135deg, #667eea, #764ba2);

/* 3. Moderate: radial gradient */
background: radial-gradient(ellipse at center, #667eea 0%, #764ba2 100%);

/* 4. Moderate: image (once loaded and decoded) */
background-image: url("pattern.png");
background-repeat: repeat;

/* 5. Slow: complex multi-stop gradient */
background: linear-gradient(
  135deg,
  #667eea 0%,
  #764ba2 25%,
  #f64f59 50%,
  #c471ed 75%,
  #12c2e9 100%
);

/* 6. Very slow: filter on background */
background: url("hero.jpg");
filter: blur(10px) brightness(0.7); /* ← rasterizes + applies GPU filter */

/* 7. Most expensive: backdrop-filter (blurs content BEHIND element) */
backdrop-filter: blur(10px) brightness(0.8);
/* Must sample and composite content behind it */
```

```css
/* ✅ Pre-render expensive backgrounds as images */
/* Instead of: complex CSS gradient (computed every repaint) */
/* Use: background-image: url('gradient.webp') (loaded once, cached) */
/* 
   The gradient.webp was pre-generated:
   - Canvas API: ctx.createLinearGradient(...)
   - CSS gradient → PNG via HTML-to-image
   - Design tool export
*/

/* ✅ Avoid background-attachment: fixed — disables layer optimization */
.hero {
  /* ❌ background-attachment: fixed prevents GPU acceleration */
  background-attachment: scroll; /* ← use this */
}
```

---

## 10. Text Rendering Optimization

Text rendering is expensive because of font loading, glyph rasterization, and anti-aliasing:

```css
/* Font loading: triggers repaint when font switches from fallback to loaded */
body {
  /* ✅ font-display: optional — use system font if web font not cached */
  /* font-display: swap — brief FOUT but users see content immediately */
  /* font-display: block — invisible text until font loaded (bad) */
}

/* ✅ Subpixel anti-aliasing (macOS/Windows) vs grayscale */
body {
  /* Default on macOS: -webkit-font-smoothing: antialiased is grayscale,
     but subpixel-antialiased is the OS default and actually sharper */
  -webkit-font-smoothing: antialiased; /* grayscale: thin text */
  /* vs: -webkit-font-smoothing: subpixel-antialiased; */
}

/* text-rendering: geometricPrecision — expensive for complex Unicode */
h1 {
  text-rendering: optimizeSpeed; /* less precise but faster */
  /* vs: text-rendering: optimizeLegibility (kerning + ligatures — slow) */
  /* vs: text-rendering: geometricPrecision (exact but slow) */
}
```

```javascript
// ✅ Cache font metrics — measureText is expensive
const fontMetricsCache = new Map();

function measureText(ctx, text, font) {
  const key = `${font}:${text}`;
  if (fontMetricsCache.has(key)) return fontMetricsCache.get(key);

  ctx.font = font;
  const metrics = ctx.measureText(text);
  fontMetricsCache.set(key, { width: metrics.width });
  return fontMetricsCache.get(key);
}
```

---

## 11. Animations Without Repaint

### The Golden Rule

```
Only animate: transform and opacity
Everything else: triggers repaint
```

### Transform-Based Animation Patterns

```css
/* ✅ Move: use transform instead of top/left */
/* ❌ */
.element {
  transition: top 0.3s;
}
/* ✅ */
.element {
  transition: transform 0.3s;
}

/* ✅ Resize: use transform: scale instead of width/height */
/* ❌ */
.button:hover {
  width: 120%;
}
/* ✅ */
.button:hover {
  transform: scale(1.2);
}

/* ✅ Show/hide: use opacity + pointer-events instead of display:none */
/* ❌ */
.menu {
  display: none; /* triggers layout + paint when toggled */
}
.menu.open {
  display: block;
}
/* ✅ */
.menu {
  opacity: 0;
  pointer-events: none;
  transform: translateY(-10px);
  transition:
    opacity 0.2s,
    transform 0.2s;
}
.menu.open {
  opacity: 1;
  pointer-events: auto;
  transform: translateY(0);
}
```

### Simulating Color Transitions Without Repaint

```css
/* ❌ background-color animation: repaint every frame */
.button {
  background-color: #4fc3f7;
  transition: background-color 0.3s;
}
.button:hover {
  background-color: #0288d1;
}

/* ✅ Overlay technique: opacity change on color layer */
.button {
  background-color: #4fc3f7;
  position: relative;
}
.button::after {
  content: "";
  position: absolute;
  inset: 0;
  background-color: #0288d1;
  opacity: 0;
  transition: opacity 0.3s; /* compositor-only */
  border-radius: inherit;
  pointer-events: none;
}
.button:hover::after {
  opacity: 1;
}
/* The "color change" is actually an opacity change on a pre-painted overlay */
```

### FLIP for Layout Animations

```javascript
// FLIP: animate between two positions using transform
// Avoids layout-triggering position changes

function flipAnimate(element, doChange) {
  // FIRST: record initial position
  const first = element.getBoundingClientRect();

  // DO the change (this triggers layout)
  doChange();

  // LAST: record final position
  const last = element.getBoundingClientRect();

  // INVERT: translate element back to initial position using transform
  const deltaX = first.left - last.left;
  const deltaY = first.top - last.top;

  element.style.transform = `translate(${deltaX}px, ${deltaY}px)`;
  element.style.transition = "none";

  // PLAY: release the invert transform — CSS transition handles animation
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      element.style.transition = "transform 0.3s cubic-bezier(0.4, 0, 0.2, 1)";
      element.style.transform = ""; // animate to natural position
    });
  });
}

// Usage: animate a card moving to a new grid position
flipAnimate(card, () => {
  grid.reorder(card, newIndex); // layout change
});
// → user sees smooth animation, no layout per frame
```

---

## 12. Contain: paint — Paint Isolation

`contain: paint` creates a paint containment boundary — elements outside the container are not repainted when the container's content changes.

```css
/* contain: paint — isolates painting to this element */
.card {
  contain: paint;
  /* When any child changes: only this card is repainted
     Siblings are NOT repainted even if they visually overlap */
}

/* ✅ Best for: repeated elements like cards, list items, widgets */
.feed-item {
  contain: paint;
  /* Each feed item is independently repainted
     Animating one item: only that item's layer repaints */
}

/* contain: strict = size + layout + style + paint */
.isolated-widget {
  contain: strict;
  width: 300px; /* required for size containment */
  height: 200px;
  /* Completely isolated: no size, layout, style, or paint
     effects escape this element */
}
```

### Containment and Overflow

```css
/* contain: paint implies overflow: hidden
   Elements outside the container's padding box are not painted */
.card {
  contain: paint;
  /* Child elements clipped to card's bounds — overflow hidden automatically */
  /* This is usually what you want */
}

/* If you need overflow visible: use contain: layout instead */
.card-with-overflow {
  contain: layout;
  /* Layout is contained, but paint is not — children can overflow */
}
```

---

## 13. Good Practices

### ✅ Profile before optimizing

```
Use DevTools Paint Flashing to see what's actually being repainted.
Don't assume — measure.

If a page is smooth: no paint optimization needed.
If there's jank: find the cause in the Performance panel.
```

### ✅ Use CSS `will-change` for elements that frequently repaint

```css
/* ✅ Pre-promote elements that will repaint/animate frequently */
.animated-icon {
  will-change: transform, opacity;
  /* Browser creates a compositor layer before the animation starts */
  /* When animation runs: compositor handles it — no repaint */
}
```

### ✅ Prefer solid backgrounds over gradients for frequently-repainted elements

```css
/* If an element repaints often (e.g., on hover with other changes): */
.frequent-repaint-target {
  background: #4fc3f7; /* solid: fast repaint */
  /* vs: background: linear-gradient(...) which is slower to rasterize */
}
```

### ✅ Split static and dynamic content into separate layers

```css
/* Static background: one layer, painted once */
.page-background {
  position: fixed;
  inset: 0;
  background: url("pattern.svg");
  z-index: -1;
  /* This layer is painted once and cached */
}

/* Dynamic content: own layer, only repaints when needed */
.interactive-overlay {
  will-change: transform, opacity;
  /* Repaints independently from the background */
}
```

---

## 14. Bad Practices

### ❌ Animating paint-triggering properties at 60fps

```css
/* ❌ Each frame: rasterize + upload to GPU */
.pulse {
  animation: pulse 1s infinite;
}
@keyframes pulse {
  0%,
  100% {
    background-color: #4fc3f7;
  }
  50% {
    background-color: #0288d1;
  }
}

/* ✅ Use opacity overlay: compositor only */
.pulse {
  background-color: #4fc3f7;
  position: relative;
}
.pulse::after {
  content: "";
  position: absolute;
  inset: 0;
  background-color: #0288d1;
  animation: pulse-opacity 1s infinite;
  border-radius: inherit;
}
@keyframes pulse-opacity {
  0%,
  100% {
    opacity: 0;
  }
  50% {
    opacity: 1;
  }
}
```

### ❌ Using `filter: blur()` on animated elements

```css
/* ❌ blur() animation: CPU repaint every frame */
.element {
  transition: filter 0.3s;
}
.element:hover {
  filter: blur(4px);
}

/* ✅ Pre-render blurred version, animate opacity */
.element-wrapper {
  position: relative;
}
.element-original {
  /* ... */
}
.element-blurred {
  position: absolute;
  inset: 0;
  filter: blur(4px); /* rendered once */
  opacity: 0;
  transition: opacity 0.3s; /* compositor only */
}
.element-wrapper:hover .element-blurred {
  opacity: 1;
}
```

### ❌ `background-attachment: fixed` on large elements

```css
/* ❌ Fixed attachment: browser must repaint on scroll */
.parallax {
  background: url("hero.jpg") fixed center/cover;
  /* Disables layer caching — must repaint on EVERY scroll pixel */
}

/* ✅ Use transform for parallax instead */
.parallax-container {
  overflow: hidden;
}
.parallax-bg {
  transform: translateY(var(--scroll-offset));
  will-change: transform; /* compositor layer */
}
/* Update --scroll-offset via JavaScript — no repaint */
```

---

## 15. Common Mistakes

### Mistake 1 — Not knowing `visibility: hidden` still triggers paint

```css
/* ❌ Assumption: visibility:hidden has no paint cost */
.tooltip {
  visibility: hidden; /* occupies space, not painted */
}
.tooltip.visible {
  visibility: visible; /* triggers paint when toggled! */
}

/* vs. opacity: 0 which is composited without repaint */
.tooltip {
  opacity: 0;
  pointer-events: none;
  /* transitions to opacity: 1 are compositor-only */
}
```

### Mistake 2 — Creating layers for non-animating elements

```css
/* ❌ will-change applied to static elements — wastes GPU memory */
.static-card {
  will-change: transform; /* never actually animated! */
  /* Wastes a GPU texture slot for this card forever */
}

/* ✅ Apply dynamically just before animation */
card.addEventListener('mouseenter', () => {
  card.style.willChange = 'transform, box-shadow';
});
card.addEventListener('mouseleave', () => {
  setTimeout(() => { card.style.willChange = 'auto'; }, 300);
});
```

### Mistake 3 — Forgetting that `border-radius` on transformed elements can be expensive

```css
/* border-radius + transform = browser may repaint (varies by implementation) */
/* Monitor with DevTools paint flashing when using both */

.card {
  border-radius: 12px;
  /* If this element has will-change: transform AND changes shape on hover,
     the repaint cost includes anti-aliased corners */
}
```

---

## 16. Interview-Level Explanation

> **"How do you optimize paint performance? What CSS properties trigger repaint?"**

**Strong answer:**

> "Paint performance comes down to: how often, how much area, and how expensive each repaint is.
>
> CSS properties divide into three categories by rendering cost. Layout-triggering properties — width, height, margin, padding, position — cause the most expensive path: layout recalculation, then paint, then composite. Paint-triggering properties — background-color, color, box-shadow, border-radius — skip layout but still require rasterization: re-computing pixels and uploading a new bitmap to the GPU. Compositor-only properties — transform and opacity — don't touch the CPU after first render; they're matrix math applied to an existing GPU texture by the compositor thread.
>
> The key optimization for animations is staying on the compositor thread. Transform and opacity animations never trigger repaint, can't be blocked by JavaScript, and run at 60fps even during heavy main thread work. Any animation on a paint-triggering property costs a repaint per frame — multiply that by 60 and it adds up.
>
> For specific expensive properties: box-shadow is particularly costly because its paint area scales with blur radius. The optimization is either to use a pre-drawn pseudo-element (`::after`) on its own layer and animate its opacity, or to use `filter: drop-shadow` which is GPU-accelerated. Background-color animations are replaced by opacity animations on a pre-painted overlay element.
>
> Compositor layers isolate repaints — when an element is on its own layer, changes to it don't contaminate other elements. `will-change: transform` or `transform: translateZ(0)` promotes an element to its own layer. But layers cost GPU memory (proportional to pixel area), so they should be applied sparingly and preferably dynamically: add before animation starts, remove after it ends.
>
> `contain: paint` is underused — it declares that nothing inside will paint outside its bounds, and nothing outside affects what's inside. Used on repeated elements like cards or list items, it scopes repaints to the individual card rather than triggering a broader repaint region."

---

## 17. Exercises

### Exercise 1 — Analyze and optimize

```css
/* This CSS causes severe paint performance issues during the animation.
   Identify ALL problems and fix them. */

.product-card {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  box-shadow: 0 10px 40px rgba(102, 126, 234, 0.8);
  border-radius: 12px;
  transition:
    background-color 0.3s,
    box-shadow 0.3s,
    transform 0.3s;
}

.product-card:hover {
  background: linear-gradient(135deg, #764ba2 0%, #667eea 100%);
  box-shadow: 0 20px 60px rgba(102, 126, 234, 0.9);
  transform: translateY(-4px) scale(1.02);
}
```

<details>
<summary>Solution</summary>

```
Problems:
1. background (gradient) transition: repaint every frame — expensive gradient rasterization
2. box-shadow transition: repaint every frame with expensive shadow calculation
3. transform transition: this one is FINE — compositor only

Optimized:

.product-card {
  position: relative;
  border-radius: 12px;
  /* Keep the original gradient as the static base */
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  /* Transform is kept — it's compositor-friendly */
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  will-change: transform; /* promote layer */
}

/* Pre-render hover gradient as a separate layer, animate opacity */
.product-card::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  background: linear-gradient(135deg, #764ba2 0%, #667eea 100%);
  opacity: 0;
  transition: opacity 0.3s; /* compositor-only */
  pointer-events: none;
}

/* Pre-render shadow as a separate layer, animate opacity */
.product-card::after {
  content: '';
  position: absolute;
  inset: -5px;
  border-radius: inherit;
  box-shadow: 0 20px 60px rgba(102, 126, 234, 0.9);
  opacity: 0;
  transition: opacity 0.3s; /* compositor-only */
  pointer-events: none;
  z-index: -1;
}

.product-card:hover {
  transform: translateY(-4px) scale(1.02); /* compositor-only */
}

.product-card:hover::before { opacity: 1; } /* compositor-only */
.product-card:hover::after  { opacity: 1; } /* compositor-only */

/*
  Before: every hover frame repaints gradient + shadow = expensive
  After: all hover animation is compositor-only = essentially free
  Cost: 2 additional GPU layers per card (acceptable)
*/
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/05-paint-repaint.md`](../browser-internals/05-paint-repaint.md) — Browser paint internals
- [`browser-internals/06-composite-layers.md`](../browser-internals/06-composite-layers.md) — Compositor layers in detail
- [`browser-internals/07-gpu-acceleration.md`](../browser-internals/07-gpu-acceleration.md) — GPU role in rendering
- [`rendering/01-dom-batching.md`](./01-dom-batching.md) — Batching DOM operations
- [`animations/03-compositor-animations.md`](../animations/03-compositor-animations.md) — Compositor-only animations

---

<div align="center">

**Next:** [`rendering/05-hydration-patterns.md`](./05-hydration-patterns.md) →

</div>
