# 03 — Compositor Animations

> **"The compositor thread is the browser's promise that some animations will never drop a frame, no matter what JavaScript is doing. Understanding what qualifies for that promise — and what silently breaks it — is the difference between an animation that's butter-smooth under load and one that stutters the moment the main thread gets busy."**

Compositor animations run on a separate thread from JavaScript, layout, and paint — meaning they continue smoothly even while the main thread is blocked by expensive computation. This document explains the rendering pipeline's threading model, exactly which animations qualify for compositor-only execution, how layer promotion works, the conditions that silently demote an animation back to the main thread, and how to verify in DevTools that your animations are actually getting the fast path.

---

## 📚 Table of Contents

1. [The Threading Model](#1-the-threading-model)
2. [What Makes an Animation Compositor-Only](#2-what-makes-an-animation-compositor-only)
3. [Layer Promotion](#3-layer-promotion)
4. [The Compositor Animation Pipeline](#4-the-compositor-animation-pipeline)
5. [Conditions That Break Compositor-Only Status](#5-conditions-that-break-compositor-only-status)
6. [will-change Deep Dive](#6-will-change-deep-dive)
7. [Verifying Compositor Execution in DevTools](#7-verifying-compositor-execution-in-devtools)
8. [Main-Thread-Blocked Scenario](#8-main-thread-blocked-scenario)
9. [Compositor Animations and React](#9-compositor-animations-and-react)
10. [Layer Memory Cost](#10-layer-memory-cost)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. The Threading Model

```
BROWSER RENDERING THREADS:

┌─────────────────────────────────────────────────────────────┐
│ MAIN THREAD                                                   │
│  - JavaScript execution                                       │
│  - Style recalculation                                        │
│  - Layout (reflow)                                            │
│  - Paint (rasterization, sometimes off-loaded to raster threads) │
│  Can be BLOCKED by: long JS tasks, expensive layout, GC pauses │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ commits layer tree + paint instructions
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ COMPOSITOR THREAD                                              │
│  - Receives layer tree from main thread                       │
│  - Applies transform/opacity animations per frame              │
│  - Combines layers into final frame                            │
│  - Handles scrolling (when not blocked by passive listeners)   │
│  Runs INDEPENDENTLY of main thread — NOT blocked by JS          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ GPU PROCESS                                                    │
│  - Final rasterization (if GPU rasterization enabled)          │
│  - Texture composition                                         │
│  - Frame presentation to display                               │
└─────────────────────────────────────────────────────────────┘

KEY INSIGHT:
  If main thread is blocked for 500ms (long JS task):
    - Compositor-only animations (transform/opacity): CONTINUE SMOOTHLY
    - Main-thread-dependent animations: FREEZE for 500ms
```

---

## 2. What Makes an Animation Compositor-Only

```
QUALIFYING PROPERTIES (compositor can animate independently):
  transform   (translate, scale, rotate, skew, matrix)
  opacity
  filter      (in most modern browser versions — GPU-accelerated filters)
  backdrop-filter (similarly GPU-accelerated)

WHY THESE QUALIFY:
  These properties don't change the element's geometry in a way that
  affects OTHER elements' layout or paint. The compositor can apply them
  as a simple per-frame transform to an already-rasterized texture.

DISQUALIFYING PROPERTIES (require main thread involvement):
  width, height, top, left, right, bottom, margin, padding  → LAYOUT
  background-color, border-color, box-shadow, color          → PAINT
  border-radius (in some browsers, partial support)          → PAINT

WHY THESE DISQUALIFY:
  Layout-triggering: changing these affects the geometry of this element
  AND potentially sibling/ancestor elements — must run on main thread
  Paint-triggering: changing these requires re-rasterizing pixels —
  the compositor doesn't have a CPU rasterizer, only the main/raster threads do
```

---

## 3. Layer Promotion

For an element to receive compositor-only animation treatment, it must first be promoted to its own compositing layer:

```css
/* Implicit layer promotion triggers: */

/* 1. 3D transforms (even a no-op one) */
.layer-1 {
  transform: translateZ(0);
}
.layer-2 {
  transform: translate3d(0, 0, 0);
}

/* 2. will-change hint */
.layer-3 {
  will-change: transform;
}

/* 3. Active CSS animation/transition on transform or opacity */
.layer-4 {
  transition: transform 0.3s;
}
/* (Promoted automatically WHILE animating; may demote after) */

/* 4. Position: fixed (often auto-promoted for scroll performance) */
.layer-5 {
  position: fixed;
}

/* 5. Video, canvas, iframe elements */
/* Always get their own layer due to independent rendering needs */

/* 6. CSS filter or backdrop-filter applied */
.layer-6 {
  filter: blur(2px);
}

/* 7. will-change: opacity with an active opacity animation */
.layer-7 {
  will-change: opacity;
}
```

### Why Layer Promotion Matters

```
WITHOUT PROMOTION:
  Element shares a layer (raster surface) with siblings/parent
  Animating transform on this element: compositor must understand
  WHICH PART of the shared raster surface moves — not possible
  → Falls back to main-thread repaint of the whole shared surface

WITH PROMOTION:
  Element gets its OWN raster surface (texture)
  Compositor can move/fade THIS texture independently
  → True compositor-only animation: zero main thread cost per frame
```

---

## 4. The Compositor Animation Pipeline

```
FRAME LIFECYCLE FOR A COMPOSITOR ANIMATION:

1. MAIN THREAD (once, at animation start):
   - Style recalculation: element gets transform/opacity animation
   - Layer promotion: element becomes its own compositing layer
   - Paint: element's content rasterized ONCE into a GPU texture
   - Commit: layer tree + animation keyframes sent to compositor thread

2. COMPOSITOR THREAD (every frame, ~60-120 times/second):
   - Read current animation progress (interpolated from keyframes)
   - Compute transform matrix or opacity value for this frame
   - Apply matrix/alpha to the EXISTING texture (no re-rasterization)
   - Composite all layers into the final frame
   - Submit frame to GPU for display

3. GPU (every frame):
   - Render the composited layers to the screen

CRITICAL: Steps 2-3 happen WITHOUT involving the main thread AT ALL
JavaScript can be fully blocked and the animation continues at 60fps
```

---

## 5. Conditions That Break Compositor-Only Status

Several conditions silently force a "compositor-only" animation back onto the main thread:

```css
/* ❌ Overlapping elements that aren't both promoted */
.animated {
  transform: translateX(0);
  transition: transform 0.3s;
  animation: slide 1s;
}
.sibling-without-promotion {
  position: absolute;
  /* If .animated overlaps this on a shared layer: forces main-thread paint
     to correctly handle the overlap during animation */
}
/* Fix: promote BOTH overlapping elements if they animate independently */
```

```javascript
// ❌ JavaScript reading layout properties during animation
function onAnimationFrame() {
  const rect = animatedElement.getBoundingClientRect(); // FORCES layout sync
  // Even though the CSS animation itself is compositor-only,
  // reading geometry forces the main thread to synchronize with
  // the compositor's current state — breaks the independence
}
```

```css
/* ❌ Clipping/masking interactions that require main-thread coordination */
.parent {
  overflow: hidden; /* clip */
}
.child-3d-transform {
  transform: translateZ(
    50px
  ); /* 3D transform that may exceed parent's clip in unexpected ways */
  /* Some browsers fall back to main-thread compositing for complex
     clip + 3D transform interactions */
}
```

```
OTHER DEMOTION TRIGGERS:
  - Too many compositor layers (GPU memory pressure → browser merges layers)
  - will-change applied to too many elements simultaneously
  - Complex blend modes (mix-blend-mode) requiring main-thread compositing
  - Some CSS filter combinations on older GPU drivers
  - Sub-pixel layer boundaries causing rasterization issues
```

---

## 6. will-change Deep Dive

```css
/* will-change: hints to the browser to prepare optimizations in advance */
.about-to-animate {
  will-change: transform, opacity;
}

/* WHAT IT DOES:
   - Browser may promote element to its own layer PREEMPTIVELY
   - Avoids the "first frame jank" of just-in-time layer promotion
   - Allocates GPU memory for the layer ahead of time
*/

/* WHEN TO USE:
   - Immediately before a known animation starts (e.g., on hover/focus)
   - NOT as a permanent blanket optimization
*/
```

```javascript
// ✅ Correct usage: apply just before animating, remove after
function animateCard(card) {
  card.style.willChange = "transform, opacity";

  const animation = card.animate(
    [
      { transform: "scale(1)", opacity: 1 },
      { transform: "scale(1.05)", opacity: 0.9 },
    ],
    { duration: 200, easing: "ease-out", fill: "forwards" },
  );

  animation.addEventListener("finish", () => {
    card.style.willChange = "auto"; // release the layer when done
  });
}

// On hover: prepare for animation, clean up on leave
card.addEventListener("mouseenter", () => {
  card.style.willChange = "transform";
});
card.addEventListener("mouseleave", () => {
  // Delay removal slightly in case mouseleave fires mid-transition-trigger
  setTimeout(() => {
    card.style.willChange = "auto";
  }, 300);
});
```

```css
/* ❌ DON'T apply will-change to many elements permanently */
.list-item {
  will-change: transform; /* applied to 500 list items = 500 GPU layers = memory exhaustion */
}

/* ❌ DON'T use will-change as a "just in case" performance boost */
* {
  will-change: transform;
} /* catastrophic — every element gets its own layer */
```

---

## 7. Verifying Compositor Execution in DevTools

```
CHROME DEVTOOLS — LAYERS PANEL:
  More tools → Layers
  Shows: 3D visualization of all compositing layers
  Click a layer: see "Compositing Reasons" (why it was promoted)
  Common reasons shown:
    "Has a CSS transition or animation for transform or opacity"
    "Has a will-change: transform"
    "Is a stacking context with backdrop-filter"

CHROME DEVTOOLS — PERFORMANCE PANEL:
  Record during animation
  Look for:
    GREEN "Composite Layers" events → compositor-only work (good)
    PURPLE "Paint" events DURING animation → main thread involved (investigate)
    "Recalculate Style" + "Layout" during animation → NOT compositor-only

  Healthy compositor animation timeline:
    [Composite Layers] [Composite Layers] [Composite Layers] ...
    (repeating every ~16ms, no Paint/Layout events interspersed)

  Unhealthy (main-thread-bound) animation timeline:
    [Recalculate Style][Layout][Paint][Composite Layers]
    [Recalculate Style][Layout][Paint][Composite Layers] ...
    (full pipeline running every frame — expensive)
```

```javascript
// Programmatic verification: check getAnimations() for running compositor animations
const animations = element.getAnimations();
animations.forEach((anim) => {
  console.log("Animation:", anim.id, "playState:", anim.playState);
  // Inspect anim.effect.getKeyframes() to verify only transform/opacity used
});
```

---

## 8. Main-Thread-Blocked Scenario

The defining test of a true compositor animation: does it survive a blocked main thread?

```javascript
// Test: start a compositor animation, then block the main thread
const box = document.querySelector(".box");

box.animate(
  [{ transform: "translateX(0)" }, { transform: "translateX(300px)" }],
  { duration: 2000, easing: "linear" },
);

// Immediately block the main thread for 1 second
const blockUntil = performance.now() + 1000;
while (performance.now() < blockUntil) {
  /* busy loop */
}

// RESULT:
// Compositor-only animation (transform): continues smoothly during the block,
// "catches up" visually with no stutter — main thread block is invisible to it
//
// If the animation were main-thread-bound (e.g., animating `left`):
// it would FREEZE for the full 1000ms, then jump/resume — visible stutter
```

```javascript
// React equivalent: verify your loading spinner doesn't freeze during heavy renders
function HeavyComponent() {
  useEffect(() => {
    // Simulate expensive synchronous work
    const start = performance.now();
    while (performance.now() - start < 500) {} // blocks main thread 500ms
  });

  return (
    <div>
      {/* If this spinner uses transform-based CSS animation: stays smooth */}
      {/* If it uses a JS-driven non-WAAPI animation: freezes during the block */}
      <Spinner />
    </div>
  );
}
```

---

## 9. Compositor Animations and React

```jsx
// ✅ CSS-driven compositor animation — survives React re-renders and heavy computation
function Spinner() {
  return <div className="spinner" />; // CSS: animation: spin 1s linear infinite;
}
/* .spinner { animation: spin 1s linear infinite; }
   @keyframes spin { to { transform: rotate(360deg); } }
   This NEVER stutters, even if React is doing heavy reconciliation work,
   because it runs entirely on the compositor thread. */

// ❌ State-driven "animation" via React re-renders — main-thread bound
function BadSpinner() {
  const [rotation, setRotation] = useState(0);
  useEffect(() => {
    const interval = setInterval(() => setRotation((r) => (r + 6) % 360), 16);
    return () => clearInterval(interval);
  }, []);
  return <div style={{ transform: `rotate(${rotation}deg)` }} />;
  /* Each frame: setState → re-render → commit → main thread work
     If main thread is busy: this spinner STUTTERS or freezes */
}

// ✅ Framer Motion uses WAAPI/compositor under the hood for transform/opacity
import { motion } from "framer-motion";
function GoodSpinner() {
  return (
    <motion.div
      animate={{ rotate: 360 }}
      transition={{ duration: 1, repeat: Infinity, ease: "linear" }}
    />
  );
  /* Framer Motion's default animations use the WAAPI / compositor path
     for transform/opacity — does not re-render React on every frame */
}
```

---

## 10. Layer Memory Cost

```
EACH COMPOSITOR LAYER CONSUMES GPU MEMORY:
  Memory ≈ width × height × 4 bytes (RGBA) × device pixel ratio²

  Example: 400×300px element at 2x DPR:
  800 × 600 × 4 bytes = 1,920,000 bytes ≈ 1.83 MB per layer

  100 such layers: 183 MB of GPU memory — can cause:
    - GPU memory pressure
    - Browser merging/demoting layers (loses compositor-only benefit)
    - Crashes on memory-constrained devices (older phones)

BUDGET GUIDANCE:
  Desktop: generally tolerant of 20-50 active layers
  Mobile: much more constrained — aim for < 10-15 active layers
  Always remove will-change after animation completes to free the layer
```

```javascript
// Monitor layer count in development
function logLayerCount() {
  // Chrome DevTools Protocol (for automated testing/monitoring)
  // In manual testing: DevTools → Layers panel → layer count shown at top
  console.log("Check DevTools > More tools > Layers for current layer count");
}

// Defensive pattern: limit simultaneous will-change elements
class WillChangeManager {
  #active = new Set();
  #maxConcurrent = 10;

  request(element) {
    if (this.#active.size >= this.#maxConcurrent) {
      // Release the oldest
      const oldest = this.#active.values().next().value;
      oldest.style.willChange = "auto";
      this.#active.delete(oldest);
    }
    element.style.willChange = "transform, opacity";
    this.#active.add(element);
  }

  release(element) {
    element.style.willChange = "auto";
    this.#active.delete(element);
  }
}
```

---

## 11. Good Practices

### ✅ Animate only transform and opacity for guaranteed smoothness

```css
/* ✅ Compositor-only — survives main thread congestion */
.smooth {
  transition:
    transform 0.3s,
    opacity 0.3s;
}
```

### ✅ Apply will-change just-in-time, remove after

```javascript
// ✅ Temporary promotion around the animation window only
el.style.willChange = "transform";
await animateElement(el);
el.style.willChange = "auto";
```

### ✅ Verify with DevTools Layers panel before shipping

```
Check: does the animated element have its own layer?
Check: compositing reason matches your expectation?
Check: Performance panel shows no Paint/Layout during the animation?
```

---

## 12. Bad Practices

### ❌ Assuming all CSS animations are compositor-only

```css
/* ❌ This is a main-thread animation, NOT compositor-only */
.bad {
  transition:
    width 0.3s,
    left 0.3s,
    background-color 0.3s;
}
/* Will stutter under main thread load — verify in DevTools, don't assume */
```

### ❌ Permanent will-change on many elements

```css
/* ❌ Every card gets a permanent layer — wastes GPU memory */
.card {
  will-change: transform;
} /* applied to a 200-card grid */
```

---

## 13. Common Mistakes

### Mistake 1 — Reading layout properties during a compositor animation

```javascript
// ❌ Breaks compositor independence by forcing synchronization
element.animate(
  [{ transform: "translateX(0)" }, { transform: "translateX(200px)" }],
  { duration: 500 },
);
requestAnimationFrame(function check() {
  const rect = element.getBoundingClientRect(); // forces main thread sync with compositor!
  requestAnimationFrame(check);
});

// ✅ Avoid layout reads during compositor animations unless necessary
// If you need position tracking: use the Animation object's currentTime
// and compute position mathematically instead of querying the DOM
```

### Mistake 2 — Forgetting mobile GPU constraints

```css
/* ❌ Desktop-tested animation with many simultaneous layers — fine on desktop,
   crashes or severely degrades on older/budget mobile devices */
.particle {
  will-change: transform;
} /* 200 particles, each own layer */

/* ✅ Test on real mobile devices, reduce layer count or use Canvas/WebGL
   for high particle counts instead of DOM elements */
```

### Mistake 3 — Not realizing `filter` animations may not be compositor-only on all browsers

```css
/* filter animations: GPU-accelerated in most modern browsers,
   but verify on your target browser matrix — older browser versions
   or specific filter combinations (e.g., feTurbulence SVG filters)
   may force main-thread rasterization */
.blur-transition {
  transition: filter 0.3s;
}
.blur-transition:hover {
  filter: blur(
    4px
  ); /* verify in DevTools Performance panel for your target browsers */
}
```

---

## 14. Interview-Level Explanation

> **"What makes an animation run on the compositor thread instead of the main thread? Why does this matter?"**

**Strong answer:**

> "The browser's rendering pipeline splits work across the main thread — which runs JavaScript, style recalculation, layout, and paint — and the compositor thread, which combines pre-rasterized layers into final frames. Compositor-only animations are ones where every frame's visual update can be expressed purely as a GPU texture transform — translation, scale, rotation, or alpha blending — without needing to re-rasterize pixels or recompute layout.
>
> This is why `transform` and `opacity` are the two properties you should animate for guaranteed smooth performance. The element gets rasterized once into a GPU texture when the animation begins, and every subsequent frame is just the compositor applying a different transform matrix or alpha value to that existing texture — work that happens entirely on the compositor thread, independent of whatever the main thread is doing. If the main thread is blocked by a long JavaScript task, a compositor animation continues at full frame rate; you simply won't be able to interact with the page, but the animation itself stays smooth.
>
> Animating `width`, `left`, `margin`, or similar layout-affecting properties requires the main thread on every frame — recalculate style, run layout, repaint, then composite. If the main thread is busy, that animation visibly stutters or freezes.
>
> For an element to get this compositor-only treatment, it needs to be promoted to its own compositing layer. This happens implicitly when you apply an active transform/opacity animation, but you can request it ahead of time with `will-change: transform` to avoid the brief jank of just-in-time layer promotion on the first frame. The tradeoff is GPU memory — each layer is roughly width times height times 4 bytes times the device pixel ratio squared, so promoting hundreds of elements can exhaust GPU memory, especially on mobile devices, and the browser may respond by merging layers back together, silently losing the compositor-only benefit.
>
> To verify an animation is actually compositor-only rather than assuming it, Chrome DevTools' Layers panel shows you the compositing reason for each layer, and the Performance panel timeline shows whether Paint and Layout events are interspersed with the animation frames — a healthy compositor animation shows only repeated 'Composite Layers' events with nothing else."

---

## 15. Exercises

### Exercise 1 — Diagnose and fix

```css
/* This card hover effect stutters when the page has heavy JavaScript running.
   Identify why, and fix it to be fully compositor-only. */

.card {
  transition:
    box-shadow 0.3s,
    width 0.3s,
    background-color 0.3s;
  width: 300px;
}

.card:hover {
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  width: 320px;
  background-color: #f5f5f5;
}
```

<details>
<summary>Solution</summary>

```
Problems:
1. width: 300px → 320px — triggers LAYOUT (most expensive)
2. background-color — triggers PAINT
3. box-shadow — triggers PAINT (and can be expensive depending on blur radius)
None of these are compositor-only — full pipeline runs every frame

Fixed (compositor-only equivalent):

.card {
  width: 300px;
  position: relative;
  transition: transform 0.3s;
}

.card:hover {
  transform: scale(1.0667); /* 320/300 ≈ 1.0667 — visually equivalent to width change */
}

/* Pre-rendered shadow as a separate layer, faded in via opacity */
.card::after {
  content: '';
  position: absolute;
  inset: -5px;
  border-radius: inherit;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  opacity: 0;
  transition: opacity 0.3s; /* compositor-only */
  pointer-events: none;
  z-index: -1;
}
.card:hover::after {
  opacity: 1; /* compositor-only */
}

/* Background color "change" via overlay + opacity */
.card::before {
  content: '';
  position: absolute;
  inset: 0;
  background-color: #f5f5f5;
  border-radius: inherit;
  opacity: 0;
  transition: opacity 0.3s; /* compositor-only */
  z-index: -1;
}
.card:hover::before {
  opacity: 1; /* compositor-only */
}

/* Result: hover effect now uses ONLY transform and opacity
   Verify in DevTools Layers panel: card and pseudo-elements should
   show "Active animation" or "will-change" as compositing reasons,
   with NO Paint/Layout events during the transition in Performance panel */
```

</details>

---

## 🔗 Related Topics

- [`rendering/04-paint-optimization.md`](../rendering/04-paint-optimization.md) — Paint cost hierarchy
- [`browser-internals/06-composite-layers.md`](../browser-internals/06-composite-layers.md) — Compositor and layer fundamentals
- [`animations/01-css-animations.md`](./01-css-animations.md) — CSS animation mechanics
- [`animations/02-javascript-animations.md`](./02-javascript-animations.md) — Web Animations API
- [`rendering/03-cooperative-scheduling.md`](../rendering/03-cooperative-scheduling.md) — Main thread scheduling

---

<div align="center">

**Next:** [`animations/04-micro-interactions.md`](./04-micro-interactions.md) →

</div>
