# 05 — Paint & Repaint

> **"Paint is where geometry becomes pixels. After layout tells the browser WHERE things are, paint tells it WHAT they look like. Understanding paint separates engineers who blindly apply `will-change` from those who know exactly which CSS properties skip it entirely."**

Paint (rasterization) is the browser stage that fills in the actual pixels for each visible element. It happens after layout and before compositing. It's more nuanced than it first appears — not all CSS properties trigger paint, some paints are more expensive than others, and the browser uses sophisticated layer and tile systems to avoid repainting the same pixels twice. This document covers what paint is, what triggers it, what makes it expensive, and how to architect your CSS and JavaScript to minimize unnecessary painting.

---

## 📚 Table of Contents

1. [What Paint Does](#1-what-paint-does)
2. [The Painting Order](#2-the-painting-order)
3. [What Triggers Paint (But Not Layout)](#3-what-triggers-paint-but-not-layout)
4. [What Skips Paint Entirely](#4-what-skips-paint-entirely)
5. [The Rasterizer — How Pixels Are Generated](#5-the-rasterizer--how-pixels-are-generated)
6. [Tiling — Painting in Chunks](#6-tiling--painting-in-chunks)
7. [Layer-Based Painting](#7-layer-based-painting)
8. [Paint Invalidation — What Gets Repainted](#8-paint-invalidation--what-gets-repainted)
9. [Expensive Paint Operations](#9-expensive-paint-operations)
10. [Detecting Paint with DevTools](#10-detecting-paint-with-devtools)
11. [CSS Properties and Their Paint Cost](#11-css-properties-and-their-paint-cost)
12. [The Paint Worklet (Houdini)](#12-the-paint-worklet-houdini)
13. [Optimizing Paint for Animations](#13-optimizing-paint-for-animations)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. What Paint Does

After layout produces geometry (positions and sizes), paint converts that geometry plus styles into **bitmaps** — grids of colored pixels for each compositor layer.

```
LAYOUT OUTPUT:                    PAINT OUTPUT:
┌──────────────────────┐          ┌──────────────────────┐
│ .header:             │          │ ████████████████████ │  ← blue pixels
│   x: 0, y: 0         │  paint   │ ██HEADER TEXT      ██ │  ← white text pixels
│   w: 1200, h: 60     │ ──────►  │ ████████████████████ │
│   background: navy   │          └──────────────────────┘
│   color: white       │          (bitmap: 1200×60 RGBA pixels)
└──────────────────────┘
```

Paint happens **per layer**. Each compositor layer gets its own bitmap. The GPU then composites (merges) all bitmaps onto the screen.

### What Paint Fills In

```
Paint rasterizes:
  ✦ Background colors and gradients
  ✦ Background images (decoded + scaled)
  ✦ Border colors and styles (solid, dashed, dotted...)
  ✦ Border radius clipping
  ✦ Text (each glyph, anti-aliased)
  ✦ Box shadows (computationally expensive)
  ✦ Text shadows
  ✦ Outlines
  ✦ SVG rendering
  ✦ Canvas 2D operations
  ✦ Images (decoded bitmaps placed in the layer)
```

---

## 2. The Painting Order

Paint doesn't just fill in pixels element by element. It follows a specific **painting order** based on the stacking context rules from layout.

### The Seven Painting Phases

For each stacking context, the browser paints in this exact order:

```
Phase 1: Background color of the element
Phase 2: Background image of the element
Phase 3: Border of the element
Phase 4: Children in normal flow (block formatting contexts)
Phase 5: Floating descendants
Phase 6: Children in normal flow (inline formatting contexts)
Phase 7: Positioned descendants (z-index order)

Then: Outline of the element
```

### Why Order Matters for Performance

The painting order means that **overlapping elements may need to be re-painted together**. If element A is below element B and A's background changes, the browser may need to repaint B's area too — because B visually overlaps A.

```
Before repaint:                After background color change on A:
┌──────────────────────┐       ┌──────────────────────┐
│ A: background:red    │  →   │ A: background:blue   │
│  ┌────────────────┐  │       │  ┌────────────────┐  │
│  │ B (overlaps A) │  │       │  │ B must repaint │  │
│  └────────────────┘  │       │  └────────────────┘  │
└──────────────────────┘       └──────────────────────┘

B's pixels are affected by A's background (visible behind B's transparency)
So B must be repainted even though B's own styles didn't change
```

This is mitigated when B is on its own compositor layer — then A and B can be painted independently and composited together.

---

## 3. What Triggers Paint (But Not Layout)

These CSS properties change how an element looks but don't affect its geometry — so they skip the layout stage and go straight to paint:

```javascript
// These trigger PAINT (not layout):
element.style.color = "red";
element.style.backgroundColor = "blue";
element.style.backgroundImage = "url(...)";
element.style.boxShadow = "0 2px 8px rgba(0,0,0,0.2)";
element.style.textShadow = "1px 1px 2px black";
element.style.borderColor = "navy"; // only color, not width
element.style.borderRadius = "8px"; // changes paint shape, not geometry
element.style.outline = "2px solid blue";
element.style.textDecoration = "underline";
element.style.caretColor = "blue";

// Visibility change:
element.style.visibility = "hidden"; // paint area preserved, pixels become transparent
```

### The Paint Pipeline for These Properties

```
JavaScript changes color:
  1. Style recalculation: new computed color stored
  2. Layout: SKIPPED (geometry unchanged)
  3. Paint: pixels for this element (and overlap area) redrawn
  4. Composite: final frame assembled

Cost: style recalculation + paint + composite
(much cheaper than layout + paint + composite)
```

---

## 4. What Skips Paint Entirely

These properties are handled by the GPU compositor — they don't trigger layout OR paint:

```javascript
// These trigger COMPOSITE ONLY (no layout, no paint):
element.style.transform = "translateX(100px)";
element.style.transform = "scale(1.5)";
element.style.transform = "rotate(45deg)";
element.style.opacity = "0.5";

// CSS filter with GPU-accelerated functions:
element.style.filter = "blur(4px)"; // GPU-composited
element.style.filter = "brightness(0.8)"; // GPU-composited
```

### Why `transform` and `opacity` Are Special

The browser maintains these properties in the **compositor thread** — separate from the main JavaScript thread. When you animate `transform` or `opacity`:

1. The compositor thread reads the animation values
2. Applies them to the layer's GPU texture (just matrix math or alpha blending)
3. Composites layers onto screen

**No JavaScript. No layout. No paint. Just GPU math.**

This means `transform`/`opacity` animations run smoothly even when:

- JavaScript is executing a long task
- Garbage collection is pausing the main thread
- Layout thrashing is happening elsewhere

```
Main thread blocked (GC, heavy JS):
  [GC PAUSE: 80ms]

Transform animation during that pause:
  [compositor thread: frame 1] [frame 2] [frame 3] [frame 4] [frame 5]
  ← smooth 60fps — compositor thread is independent of main thread
```

---

## 5. The Rasterizer — How Pixels Are Generated

The rasterizer converts vector instructions (draw a rectangle here, draw text there) into pixel data.

### Rasterization Approaches

Modern browsers use two rasterization approaches:

**CPU Rasterization (Software)**

```
CPU processes drawing commands:
  - Fills pixels one-by-one based on geometry
  - Handles complex paths (SVG, border-radius, gradients)
  - Result stored in system RAM, uploaded to GPU for compositing
  - Flexible but relatively slow for complex operations
```

**GPU Rasterization (Hardware)**

```
GPU processes drawing commands:
  - Graphics card accelerates pixel fill
  - Very fast for GPU-friendly operations (rectangles, gradients)
  - Less flexible for very complex paths
  - Chrome uses GPU rasterization by default on supported hardware
```

### Rasterization Threads

Modern browsers rasterize on background threads, not the main thread:

```
Main thread: triggers paint, sends display list to rasterizer
Raster thread 1: rasterizes tiles in viewport area
Raster thread 2: rasterizes nearby tiles (pre-rasterization)
Raster thread N: ...

This means paint itself doesn't block the main thread in modern browsers.
The main thread creates the "display list" (what to paint), then
background raster threads do the actual pixel work.
```

---

## 6. Tiling — Painting in Chunks

The browser doesn't paint an entire layer as one giant bitmap. Instead, it divides layers into **tiles** and rasterizes them independently.

### Why Tiles?

```
Page scroll area: 3000px tall
Viewport: 900px tall

Without tiles:
  - On page load: rasterize all 3000px
  - Memory: huge
  - Time: slow initial paint

With tiles (e.g., 256×256px tiles):
  - On page load: rasterize only visible tiles + nearby tiles
  - Rest of page: rasterized on-demand as user scrolls
  - Memory: much smaller
  - Response: faster initial paint
```

### Tile Priority

```
Tiles are prioritized by:
  1. Viewport tiles (highest priority — user sees these)
  2. Tiles near the viewport (pre-rasterized for smooth scroll)
  3. Far-off tiles (low priority, rasterized when idle)

On scroll:
  - New tiles entering viewport are promoted to high priority
  - Tiles scrolled far away are de-prioritized (may be evicted from GPU memory)
```

### Checkerboard Pattern

If tiles aren't rasterized in time (fast scroll, low-end device), you see the **checkerboard pattern** — the gray/white placeholder tiles the browser shows when content isn't ready.

```
Fast scroll on a slow device:

[Rasterized content] [checkerboard] [checkerboard]
                      ↑ tiles not ready yet
                      user sees gray squares briefly
                      browser rasterizes as fast as it can
```

---

## 7. Layer-Based Painting

Each **compositor layer** has its own bitmap. Elements on separate layers can be painted independently — changing one doesn't require repainting the other.

### Why Layers Exist

```
Without layers (single bitmap):
  When the sidebar updates:
    1. Clear entire page bitmap
    2. Repaint everything (sidebar + content + header)

With layers (separate bitmaps):
  When the sidebar updates:
    1. Repaint only the sidebar layer
    2. Composite sidebar + content + header
    (content and header layers are unchanged — no repaint needed)
```

### What Gets Its Own Layer

```css
/* These create compositor layers: */
will-change: transform | opacity;  /* explicit promotion */
transform: translateZ(0);          /* old hack, still works */
transform: translate3d(0,0,0);    /* same hack */
position: fixed;                   /* always on own layer */
video                              /* always on own layer */
canvas                             /* always on own layer */
WebGL                              /* always on own layer */
iframe                             /* often on own layer */
```

### Layer Memory Cost

Each layer consumes GPU memory equal to its pixel area × 4 bytes (RGBA):

```
Layer size examples:
  100×100px  layer: 100 × 100 × 4 = 40KB
  800×600px  layer: 800 × 600 × 4 = 1.92MB
  1920×1080px layer: 1920 × 1080 × 4 = 8.29MB
  Full HD page (6000px tall): 1920 × 6000 × 4 = 46.08MB
```

Promoting many elements to layers can exhaust GPU memory on mobile devices, causing performance to degrade badly.

### Layer Explosion Anti-Pattern

```css
/* ❌ Promotes every list item to its own layer */
.list-item {
  will-change: transform; /* applied unconditionally */
}
/* With 1000 items: 1000 × ~500KB = ~500MB GPU memory */
/* Far worse than not having layers at all */

/* ✅ Only promote the item actively being animated */
.list-item.is-animating {
  will-change: transform; /* added via JavaScript when animation starts */
}
/* Remove after animation completes */
.list-item {
  will-change: auto; /* fallback */
}
```

---

## 8. Paint Invalidation — What Gets Repainted

When a property that triggers paint changes, the browser marks the affected area as **paint invalid** and schedules a repaint. Understanding what gets marked invalid helps you minimize repaint scope.

### Invalidation Scope

```
Scenario 1: Single element background color change
  → Invalidates: the element's background area
  → Repaints: that element's background + any visually overlapping elements
  → Other areas: unchanged

Scenario 2: Box shadow change
  → Invalidates: element + shadow area (extends outside border box)
  → Box shadows can invalidate larger areas than the element itself

Scenario 3: Text color change
  → Invalidates: text area within the element
  → Repaints: text glyphs only (not background)

Scenario 4: Element added to / removed from DOM
  → Invalidates: element's area + all overlapping elements
  → May trigger layout (DOM change) → then full paint of dirty area
```

### Paint Containment

```css
.widget {
  contain: paint;
  /* PROMISE: this element's paint cannot extend outside its border box
     No overflow painting, no box shadows extending outside
     Browser can clip paint to the border box exactly */
}

/* Practical benefit: changing anything inside .widget
   only repaints inside .widget — nothing outside is invalidated */
```

---

## 9. Expensive Paint Operations

Not all paints are equal. Some CSS properties require significantly more computation to rasterize.

### Paint Cost Ranking

```
CHEAPEST → MOST EXPENSIVE

Solid color background:    O(1) — fill with one color
Simple gradient:           O(area) — compute gradient per pixel
Text (cached):             O(glyph count) — lookup glyph bitmaps
Text (new font/size):      O(glyph count × AA) — rasterize each glyph
Border (solid):            O(perimeter) — draw border lines
Border-radius:             O(corner complexity) — curved fill algorithms
Background-image:          O(decode + scale) — image decoding expensive
Box shadow (small blur):   O(shadow area)
Box shadow (large blur):   O(shadow area × blur radius) — EXPENSIVE
Text shadow:               O(text area × blur) — EXPENSIVE
Filter: blur():            O(element area × radius²) — VERY EXPENSIVE
Multiple backgrounds:      O(layers × area)
SVG with complex paths:    O(path complexity)
```

### Box Shadow Performance

```css
/* ❌ Large blur radius — expensive to rasterize */
.card {
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.5); /* blur: 40px — large spread */
}

/* ✅ Smaller blur radius — much cheaper */
.card {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15); /* blur: 8px — fine */
}

/* ✅✅ Use ::after with transform for "fake" shadow — composite only for animation */
.card::after {
  content: "";
  position: absolute;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
  opacity: 0;
  transition: opacity 0.3s; /* animate opacity — composite only! */
}
.card:hover::after {
  opacity: 1;
}
/* The shadow is painted once, then faded with opacity (composite only) */
/* Far cheaper than animating box-shadow directly */
```

### Filter Performance

```css
/* ❌ CSS filter on large elements — expensive repaint */
.large-image:hover {
  filter: blur(4px); /* rasterizes entire image + blur pass */
}

/* ✅ Apply filter to a small pseudo-element overlay instead */
/* Or: pre-blur the image as a separate static asset */

/* ✅ GPU-composited filters (no paint): */
.element {
  filter: brightness(0.8); /* GPU, fast */
  filter: contrast(1.2); /* GPU, fast */
  filter: saturate(0.5); /* GPU, fast */
  filter: opacity(0.9); /* use `opacity` property instead — same GPU cost */
}

/* ❌ CPU-rasterized filters (paint): */
.element {
  filter: blur(4px); /* slow — Gaussian blur is expensive */
  filter: drop-shadow(...); /* similar to box-shadow cost */
}
```

---

## 10. Detecting Paint with DevTools

### Paint Flashing

Chrome DevTools can highlight areas being repainted in real-time:

```
DevTools → ⋮ Menu → More Tools → Rendering → Paint Flashing

Green overlay appears on any area being repainted on the current frame.

What to look for:
  ✅ Minimal green — only changed areas repaint
  ❌ Full-page green on every frame — entire page repainting (bad)
  ❌ Unexpected areas painting — neighboring elements caught in repaint
```

### Layer Borders

```
DevTools → Rendering → Layer Borders

Orange lines: compositor layer boundaries
Blue dotted lines: tile boundaries within a layer

Use this to:
  - Verify which elements are on their own layer
  - Detect layer explosion (too many orange boxes)
  - Confirm will-change is creating layers as expected
```

### Paint Profiler

```
Performance tab → Record → Stop → Find a "Paint" event in flame graph
  → Click the Paint event → "Paint Profiler" section appears

Shows:
  - Recorded paint commands (draw rectangle, draw text, etc.)
  - Which commands took the most time
  - The exact area painted
```

### Identifying Repaint Causes

```javascript
// Use PerformanceObserver to track paint timing
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === "paint") {
      console.log(entry.name, entry.startTime.toFixed(1) + "ms");
    }
  }
});
observer.observe({ entryTypes: ["paint"] });

// Entries:
// 'first-paint': when anything is first painted
// 'first-contentful-paint': when first text/image is painted
```

---

## 11. CSS Properties and Their Paint Cost

A practical reference for choosing CSS properties with paint cost in mind:

### Full Reference Table

| Property                    | Layout? | Paint? | Composite? | Notes                       |
| --------------------------- | :-----: | :----: | :--------: | --------------------------- |
| `background-color`          |   ❌    |   ✅   |     ✅     | Simple fill                 |
| `background-image` (static) |   ❌    |   ✅   |     ✅     | Image decode + blit         |
| `color`                     |   ❌    |   ✅   |     ✅     | Affects text glyphs         |
| `border-color`              |   ❌    |   ✅   |     ✅     | Skip layout                 |
| `border-radius`             |   ❌    |   ✅   |     ✅     | Clipping curves             |
| `box-shadow`                |   ❌    |   ✅   |     ✅     | ⚠️ Expensive for large blur |
| `text-shadow`               |   ❌    |   ✅   |     ✅     | ⚠️ Expensive                |
| `outline`                   |   ❌    |   ✅   |     ✅     | Similar to border           |
| `visibility: hidden`        |   ❌    |   ✅   |     ✅     | Still occupies space        |
| `filter: blur()`            |   ❌    |   ✅   |     ✅     | ⚠️ Very expensive           |
| `filter: brightness()`      |   ❌    |   ❌   |     ✅     | GPU-composited              |
| **`opacity`**               |   ❌    |   ❌   |     ✅     | ⭐ GPU only                 |
| **`transform`**             |   ❌    |   ❌   |     ✅     | ⭐ GPU only                 |
| `width`, `height`           |   ✅    |   ✅   |     ✅     | Full pipeline               |
| `padding`, `margin`         |   ✅    |   ✅   |     ✅     | Full pipeline               |
| `display`                   |   ✅    |   ✅   |     ✅     | Full pipeline               |
| `font-size`                 |   ✅    |   ✅   |     ✅     | Full pipeline               |

---

## 12. The Paint Worklet (Houdini)

CSS Houdini's Paint API lets you write custom paint logic in JavaScript that runs during the browser's paint phase. This enables effects that CSS alone cannot express.

```javascript
// my-paint-worklet.js — runs in a PaintWorklet context
registerPaint(
  "checkerboard",
  class {
    static get inputProperties() {
      return ["--checkerboard-size", "--checkerboard-color"];
    }

    paint(ctx, geometry, properties) {
      const size = parseInt(properties.get("--checkerboard-size")) || 20;
      const color = properties.get("--checkerboard-color").toString() || "#ccc";

      ctx.fillStyle = color;

      for (let y = 0; y < geometry.height / size; y++) {
        for (let x = 0; x < geometry.width / size; x++) {
          if ((x + y) % 2 === 0) {
            ctx.fillRect(x * size, y * size, size, size);
          }
        }
      }
    }
  },
);
```

```css
/* Use in CSS */
.element {
  --checkerboard-size: 30;
  --checkerboard-color: #ddd;
  background: paint(checkerboard);
}
```

```javascript
// Register the worklet
CSS.paintWorklet.addModule("/my-paint-worklet.js");
```

### Why Paint Worklets Matter

- Custom backgrounds without canvas elements or images
- Procedural effects that update via CSS custom properties
- No DOM access — runs in isolation → no blocking the main thread
- Can be animated by changing custom properties (CSS transitions work)

---

## 13. Optimizing Paint for Animations

### The Golden Rule

> **Never trigger paint in an animation. Only use `transform` and `opacity`.**

```css
/* ❌ Paint-triggering animation — jank on every frame */
@keyframes highlight {
  0% {
    background-color: white;
  }
  50% {
    background-color: yellow;
  }
  100% {
    background-color: white;
  }
}

/* ✅ Opacity-based alternative — composite only */
.item {
  position: relative;
}
.item::after {
  content: "";
  position: absolute;
  inset: 0;
  background: yellow;
  opacity: 0;
  transition: opacity 0.3s;
}
.item.highlighted::after {
  opacity: 1;
}
```

### Pre-Paint the Shadow, Fade with Opacity

```css
/* ❌ Animating box-shadow — paint every frame */
.card {
  transition: box-shadow 0.3s ease;
}
.card:hover {
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
}

/* ✅ Paint shadow once, animate opacity — composite only */
.card {
  position: relative;
}
.card::after {
  content: "";
  position: absolute;
  inset: 0;
  border-radius: inherit;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3); /* painted once */
  opacity: 0;
  transition: opacity 0.3s ease; /* opacity = composite only */
}
.card:hover::after {
  opacity: 1;
}
```

### Color Change Without Paint (Sort of)

You can't truly avoid paint for color changes — but you can minimize it:

```css
/* ❌ Animating color — paint every frame */
.button {
  transition: background-color 0.3s;
}

/* ✅ Use an overlay with opacity — composite only */
.button {
  position: relative;
  background: blue;
}
.button::after {
  content: "";
  position: absolute;
  inset: 0;
  background: darkblue; /* painted once on load */
  opacity: 0;
  transition: opacity 0.3s;
  pointer-events: none;
}
.button:hover::after {
  opacity: 1; /* composite only */
}
```

### Promoting Elements Before Animation

```javascript
// ✅ Promote to layer before animation starts, demote after
function animateElement(element) {
  // Promote (create GPU layer)
  element.style.willChange = "transform";

  const animation = element.animate(
    [{ transform: "translateX(0)" }, { transform: "translateX(200px)" }],
    { duration: 500, easing: "ease-out" },
  );

  animation.finished.then(() => {
    // Demote (release GPU memory)
    element.style.willChange = "auto";
  });
}
```

---

## 14. Good Practices

### ✅ Animate `transform` and `opacity` only

```css
/* ✅ All composite-only animations */
.fade-in {
  animation: fadeIn 0.3s ease forwards;
}
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

### ✅ Use `will-change` only when needed and clean up

```javascript
// ✅ Add before animation, remove after
element.addEventListener("mouseenter", () => {
  element.style.willChange = "transform";
});

element.addEventListener("animationend", () => {
  element.style.willChange = "auto";
});
```

### ✅ Use `contain: paint` to limit repaint scope

```css
/* ✅ Changes inside .widget don't trigger repaint outside it */
.widget {
  contain: paint;
  overflow: hidden; /* required for paint containment */
}
```

### ✅ Keep animated elements on their own layer

```css
/* ✅ Layer already created — no promotion cost during animation */
.notification {
  will-change: transform; /* always animated when it appears */
  /* justify: it's always animated, so always-on layer is appropriate */
}
```

### ✅ Use Paint Flashing during development

Always check Paint Flashing when adding new animations or interactive elements. Any unexpected green means unintended paint.

---

## 15. Bad Practices

### ❌ Animating paint-triggering properties

```css
/* ❌ Paint every frame — guaranteed jank at scale */
@keyframes pulse {
  from {
    box-shadow: 0 0 0 rgba(0, 120, 255, 0.5);
  }
  to {
    box-shadow: 0 0 20px rgba(0, 120, 255, 0);
  }
}

/* ❌ Color transition — paints every frame */
.btn {
  transition: background-color 0.2s;
}
```

### ❌ Large blur filters on animated elements

```css
/* ❌ blur() is one of the most expensive paint operations */
.background:hover {
  filter: blur(20px); /* expensive blur + repaint on hover */
}
```

### ❌ `will-change` on everything

```css
/* ❌ Creates GPU layers for everything — memory exhaustion */
* {
  will-change: transform;
}

/* ❌ On many items unconditionally */
.list-item {
  will-change: opacity;
} /* 1000 items = 1000 GPU textures */
```

### ❌ `transition: all`

```css
/* ❌ Monitors ALL CSS properties for changes
      Transitions even properties you're not intentionally changing
      May trigger paint when non-visual properties change */
.card {
  transition: all 0.3s;
}

/* ✅ Be explicit */
.card {
  transition:
    transform 0.3s,
    opacity 0.3s;
}
```

---

## 16. Common Mistakes

### Mistake 1 — Thinking `visibility: hidden` skips paint

```css
/* visibility: hidden still occupies space AND still paints */
/* The element paints transparent pixels */
/* Use display: none to remove from paint entirely */

.element {
  visibility: hidden;
} /* paints transparent — still in paint */
.element {
  display: none;
} /* removed from render tree — no paint */
.element {
  opacity: 0;
} /* paints transparent, GPU composited — cheapest */
```

### Mistake 2 — `filter: opacity()` vs `opacity`

```css
/* Both make element transparent, but different pipelines: */

opacity: 0.5;
/* Creates stacking context. GPU-composited. Fast. */

filter: opacity(0.5);
/* Also creates stacking context. But: filter pipeline.
   May cause repaint in some browsers. Use `opacity` instead. */
```

### Mistake 3 — Assuming `transform` always skips paint

```css
/* transform: skips paint ONLY if the element is on its own compositor layer */

/* If the element is NOT on its own layer (no will-change, no transform3d trick): */
.element {
  transform: translateX(100px);
}
/* This still may trigger a repaint of the containing layer */

/* Only truly skips paint when on its OWN compositor layer: */
.element {
  will-change: transform; /* ensures own layer */
  transform: translateX(100px); /* now truly paint-free */
}
```

### Mistake 4 — Not checking Paint Flashing after CSS changes

Many developers add CSS, test functionality, and ship — without ever checking if their new styles are causing unexpected repaints. Paint Flashing during testing catches these before they ship.

---

## 17. Interview-Level Explanation

> **"What is paint/rasterization? What triggers it? How do you optimize for it?"**

**Strong answer:**

> "Paint — or rasterization — is the browser stage that converts styled geometry into actual pixels. After layout determines where everything is, paint fills in what it looks like: colors, text, borders, images, shadows. It produces bitmaps for each compositor layer, which the GPU then composites onto the screen.
>
> Paint is triggered by any CSS change that affects visual appearance without affecting geometry — `color`, `background-color`, `border-color`, `box-shadow`, `border-radius`. Layout-triggering changes (width, height, position) also trigger paint since they change what needs to be drawn.
>
> Two properties skip paint entirely: `transform` and `opacity`. These are handled by the GPU compositor in a completely separate thread from JavaScript. That's why `transform` and `opacity` animations stay smooth even when the main thread is busy — they can't be blocked by your JavaScript.
>
> The most important paint optimization is to never animate paint-triggering properties. Instead of animating `background-color` for a hover state, use an `::after` pseudo-element with `opacity: 0` that contains the hover color, then animate `opacity` to 1 — composite only. Similarly, instead of animating `box-shadow`, paint the shadow onto a `::after` element and animate its opacity.
>
> For detecting paint issues, Paint Flashing in Chrome DevTools shows green overlays on everything being repainted each frame. If you're seeing unexpected green areas, use Layer Borders to understand the layer structure. The Paint Profiler in the Performance tab shows exactly which CSS properties are costing the most in each paint event.
>
> `will-change: transform` tells the browser to create a GPU layer for an element ahead of time, so there's no promotion cost when animation starts. But it consumes GPU memory — apply it only to elements you know will animate, and remove it when they stop."

---

## 18. Exercises

### Exercise 1 — Classify CSS properties

For each CSS change below, identify the rendering pipeline stages triggered (Layout / Paint / Composite):

```
a) element.style.color = 'red'
b) element.style.width = '200px'
c) element.style.transform = 'scale(1.2)'
d) element.style.boxShadow = '0 4px 12px rgba(0,0,0,0.3)'
e) element.style.opacity = '0.8'
f) element.style.display = 'none'
g) element.style.borderRadius = '50%'
h) element.style.filter = 'blur(4px)'
i) element.style.visibility = 'hidden'
j) element.style.willChange = 'transform' (applied for first time)
```

<details>
<summary>Answers</summary>

```
a) color: red             → Paint + Composite
b) width: 200px           → Layout + Paint + Composite
c) transform: scale(1.2)  → Composite only ⭐ (if on own layer)
d) box-shadow             → Paint + Composite (expensive paint)
e) opacity: 0.8           → Composite only ⭐
f) display: none          → Layout + Paint + Composite (removes from render tree)
g) border-radius: 50%     → Paint + Composite (no layout change)
h) filter: blur(4px)      → Paint + Composite (expensive!)
i) visibility: hidden     → Paint + Composite (transparent pixels painted)
j) will-change: transform → Creates compositor layer (one-time promotion cost)
```

</details>

---

### Exercise 2 — Convert to composite-only animation

Rewrite this animation to avoid triggering paint on every frame:

```css
/* ❌ Paint every frame */
@keyframes ripple {
  0% {
    box-shadow: 0 0 0 0 rgba(0, 120, 255, 0.7);
  }
  100% {
    box-shadow: 0 0 0 20px rgba(0, 120, 255, 0);
  }
}

.button:active {
  animation: ripple 0.6s ease-out;
}
```

<details>
<summary>Solution</summary>

```css
/* ✅ Composite-only ripple using transform + opacity */
.button {
  position: relative;
  overflow: hidden;
}

.button::after {
  content: "";
  position: absolute;
  /* Center the ripple */
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  border-radius: 50%;
  background: rgba(0, 120, 255, 0.4);
  transform: translate(-50%, -50%) scale(0);
  opacity: 1;
  pointer-events: none;
}

.button:active::after {
  animation: ripple 0.6s ease-out forwards;
}

@keyframes ripple {
  0% {
    width: 0;
    height: 0;
    transform: translate(-50%, -50%) scale(0);
    opacity: 1;
  }
  100% {
    width: 200px;
    height: 200px;
    transform: translate(-50%, -50%) scale(1);
    opacity: 0;
  }
}

/* This still uses width/height which triggers layout/paint.
   For truly composite-only: use transform scale instead: */

@keyframes ripple-pure {
  0% {
    transform: translate(-50%, -50%) scale(0);
    opacity: 1;
  }
  100% {
    transform: translate(-50%, -50%) scale(10);
    opacity: 0;
  }
}

/* With width/height fixed to a circle matching button size —
   only transform and opacity animate = composite only */
```

</details>

---

### Exercise 3 — Use DevTools to find repaints

1. Open Chrome DevTools
2. Navigate to any website with animations (or use your own project)
3. Go to: DevTools → ⋮ → More Tools → Rendering → Enable "Paint Flashing"
4. Hover over interactive elements, scroll the page, trigger animations

Record your observations:

- Which elements flash on hover?
- Is the entire page repainting on scroll, or just parts of it?
- Are there any unexpected repaints happening on areas you didn't change?
- Try clicking "Layer Borders" — how many compositor layers are there?

Use these findings to identify at least one repaint you could eliminate by switching to `transform`/`opacity`.

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](./01-rendering-pipeline.md) — Full pipeline overview
- [`browser-internals/04-layout-reflow.md`](./04-layout-reflow.md) — Layout before paint
- [`browser-internals/06-composite-layers.md`](./06-composite-layers.md) — Compositing after paint
- [`browser-internals/07-gpu-acceleration.md`](./07-gpu-acceleration.md) — GPU layer mechanics
- [`performance/04-raf-optimization.md`](../performance/04-raf-optimization.md) — Paint-free animation patterns
- [`animations/03-compositor-animations.md`](../animations/03-compositor-animations.md) — Compositor-only animation guide

---

<div align="center">

**Next:** [`browser-internals/06-composite-layers.md`](./06-composite-layers.md) →

</div>
