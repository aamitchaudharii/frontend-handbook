# 04 — requestAnimationFrame Optimization

> **"requestAnimationFrame is not just a timer. It's a contract with the browser: 'I have visual work to do — tell me when it's safe to do it.' That contract is what makes the difference between animations that jank and animations that fly."**

`requestAnimationFrame` (rAF) is the foundational API for all browser animation and rendering work. Used correctly, it synchronizes your JavaScript to the display's refresh cycle, guarantees pre-paint execution, and enables smooth 60fps (or 120fps) animations. Used incorrectly, it creates callback accumulation, scheduling problems, and animations that look smooth but waste CPU cycles. This document covers every nuance.

---

## 📚 Table of Contents

1. [What requestAnimationFrame Actually Is](#1-what-requestanimationframe-actually-is)
2. [The rAF Timestamp — Precision and Sync](#2-the-raf-timestamp--precision-and-sync)
3. [The Animation Loop Pattern](#3-the-animation-loop-pattern)
4. [rAF vs setTimeout — The Real Difference](#4-raf-vs-settimeout--the-real-difference)
5. [Frame Budgeting with rAF](#5-frame-budgeting-with-raf)
6. [Cancelling rAF — Memory and CPU Safety](#6-cancelling-raf--memory-and-cpu-safety)
7. [Multiple Animations — One Loop vs Many](#7-multiple-animations--one-loop-vs-many)
8. [rAF and Layout Reads — The Correct Pattern](#8-raf-and-layout-reads--the-correct-pattern)
9. [rAF Throttling for Non-Visual Work](#9-raf-throttling-for-non-visual-work)
10. [requestIdleCallback — The Complement](#10-requestidlecallback--the-complement)
11. [rAF in Web Workers](#11-raf-in-web-workers)
12. [Debugging Animation Performance](#12-debugging-animation-performance)
13. [Common Animation Patterns](#13-common-animation-patterns)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. What requestAnimationFrame Actually Is

`requestAnimationFrame` is a browser API that schedules a callback to run **before the next repaint**. It is the correct way to animate anything visual in the browser.

```
Event loop iteration with rAF:

1. Execute macrotask (e.g., event handler)
2. Drain microtask queue (Promises, etc.)
3. Rendering checkpoint:
   a. Resize/scroll observers
   b. ┌────────────────────────────────┐
      │  requestAnimationFrame         │ ← YOUR CALLBACK RUNS HERE
      │  callbacks run here            │
      └────────────────────────────────┘
   c. Style recalculation
   d. Layout
   e. Paint
   f. Composite → screen
4. Next macrotask
```

**Key insight:** rAF runs **before** paint, **after** the browser has processed input events. This means:

- Your animation changes are applied in the current frame (not the next one)
- The browser won't paint between the callback running and the frame rendering
- You're synchronized to the display's actual refresh rate

### What Makes rAF Special

```javascript
// setTimeout: runs at any point in the event loop
// May run: mid-frame, between paints, at irregular intervals
setTimeout(() => {
  element.style.transform = `translateX(${x}px)`;
}, 16); // "16ms" is approximate and not synced to display

// rAF: runs exactly once per render frame, before paint
requestAnimationFrame((timestamp) => {
  element.style.transform = `translateX(${x}px)`;
  // This change is guaranteed to appear in THIS frame
});
```

---

## 2. The rAF Timestamp — Precision and Sync

Every rAF callback receives a `DOMHighResTimeStamp` — milliseconds since page load with sub-millisecond precision.

```javascript
requestAnimationFrame((timestamp) => {
  // timestamp: ms since page navigation started
  // e.g., 1234.567890 ms
  // Precision: sub-microsecond in modern browsers
  console.log("Frame at:", timestamp.toFixed(3), "ms");
});
```

### Using the Timestamp for Frame-Rate-Independent Animation

```javascript
// ❌ Frame-rate dependent: animates faster on 120fps displays
function animate() {
  x += 5; // always moves 5px per frame — different speed at 30fps vs 120fps
  element.style.transform = `translateX(${x}px)`;
  requestAnimationFrame(animate);
}

// ✅ Frame-rate independent: velocity in pixels per second
const VELOCITY = 200; // 200px per second
let lastTime = null;

function animate(timestamp) {
  if (lastTime === null) {
    lastTime = timestamp;
    requestAnimationFrame(animate);
    return;
  }

  const delta = (timestamp - lastTime) / 1000; // convert ms to seconds
  x += VELOCITY * delta; // 200px/s × elapsed seconds

  element.style.transform = `translateX(${x}px)`;
  lastTime = timestamp;

  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);

// At 60fps: delta ≈ 0.0167s → moves 200 × 0.0167 = 3.33px per frame
// At 120fps: delta ≈ 0.0083s → moves 200 × 0.0083 = 1.67px per frame
// Total distance per second: always 200px regardless of frame rate
```

### Handling Large Delta (Tab Focus Return)

```javascript
const MAX_DELTA = 0.1; // cap at 100ms to prevent huge jumps

function animate(timestamp) {
  const rawDelta = (timestamp - lastTime) / 1000;
  const delta = Math.min(rawDelta, MAX_DELTA); // cap large deltas

  // When tab regains focus: timestamp jumps significantly
  // Without cap: animation would skip forward dramatically
  // With cap: animation resumes smoothly from current position

  x += VELOCITY * delta;
  element.style.transform = `translateX(${x}px)`;
  lastTime = timestamp;
  requestAnimationFrame(animate);
}
```

---

## 3. The Animation Loop Pattern

### Basic Loop

```javascript
class Animator {
  #rafId = null;
  #running = false;
  #lastTime = null;

  start() {
    if (this.#running) return;
    this.#running = true;
    this.#lastTime = null;
    this.#rafId = requestAnimationFrame((t) => this.#loop(t));
  }

  stop() {
    if (!this.#running) return;
    this.#running = false;
    if (this.#rafId !== null) {
      cancelAnimationFrame(this.#rafId);
      this.#rafId = null;
    }
    this.#lastTime = null;
  }

  #loop(timestamp) {
    if (!this.#running) return;

    const delta =
      this.#lastTime === null
        ? 0
        : Math.min((timestamp - this.#lastTime) / 1000, 0.1); // cap delta

    this.update(delta, timestamp);

    this.#lastTime = timestamp;
    this.#rafId = requestAnimationFrame((t) => this.#loop(t));
  }

  // Override in subclass
  update(delta, timestamp) {}
}

// Usage
class ParticleSystem extends Animator {
  constructor(canvas) {
    super();
    this.ctx = canvas.getContext("2d");
    this.particles = [];
  }

  update(delta, timestamp) {
    const { ctx, particles } = this;
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

    particles.forEach((p) => {
      p.x += p.vx * delta;
      p.y += p.vy * delta;
      p.life -= delta;

      ctx.beginPath();
      ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
      ctx.fillStyle = `rgba(${p.color}, ${p.life})`;
      ctx.fill();
    });

    // Remove dead particles
    this.particles = particles.filter((p) => p.life > 0);
  }
}
```

### Progress-Based Animation (Easing)

```javascript
function animateValue(options) {
  const {
    from,
    to,
    duration, // ms
    easing = (t) => t, // linear by default
    onUpdate,
    onComplete,
  } = options;

  let startTime = null;

  function step(timestamp) {
    if (startTime === null) startTime = timestamp;

    const elapsed = timestamp - startTime;
    const progress = Math.min(elapsed / duration, 1); // 0 to 1
    const eased = easing(progress); // apply easing
    const value = from + (to - from) * eased; // interpolate

    onUpdate(value);

    if (progress < 1) {
      requestAnimationFrame(step);
    } else {
      onComplete?.();
    }
  }

  requestAnimationFrame(step);
}

// Easing functions
const Easing = {
  linear: (t) => t,
  easeIn: (t) => t * t,
  easeOut: (t) => t * (2 - t),
  easeInOut: (t) => (t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t),
  bounce: (t) => {
    if (t < 0.364) return 7.5625 * t * t;
    if (t < 0.727) return 7.5625 * (t -= 0.545) * t + 0.75;
    if (t < 0.909) return 7.5625 * (t -= 0.818) * t + 0.9375;
    return 7.5625 * (t -= 0.955) * t + 0.984375;
  },
};

// Usage
animateValue({
  from: 0,
  to: 300,
  duration: 500,
  easing: Easing.easeOut,
  onUpdate: (value) => {
    element.style.transform = `translateX(${value}px)`;
  },
  onComplete: () => console.log("Animation done"),
});
```

---

## 4. rAF vs setTimeout — The Real Difference

```
setTimeout(fn, 16):
  ✗ Not synced to display refresh rate
  ✗ Minimum ~4ms clamp for nested timers
  ✗ Fires AFTER rendering checkpoint (mid-frame)
  ✗ Not paused in background tabs (drains battery)
  ✗ Can fire twice in one frame or skip a frame entirely
  ✓ Can be cancelled with clearTimeout
  ✓ Works for non-visual deferred work

requestAnimationFrame(fn):
  ✓ Synced to display vsync (exactly 60fps, 120fps, etc.)
  ✓ Fires BEFORE paint (changes are visible in THIS frame)
  ✓ Paused in background tabs (battery efficient)
  ✓ One call per frame — no duplicates or skips
  ✓ Receives precise timestamp
  ✓ Can be cancelled with cancelAnimationFrame
  ✗ Not suitable for non-visual work (wastes frame budget)
```

### Visual Demonstration of the Difference

```javascript
// Timeline comparison:

// setTimeout(fn, 16) timeline:
// Frame 1: [JS] [Layout] [Paint] → screen
// Frame 2: [JS] [setTimeout fires mid-frame] [Layout] [Paint] → screen
//          ↑ BAD: setTimeout fires AFTER some rendering, before the next

// rAF timeline:
// Frame 1: [JS] [rAF fires] [Layout] [Paint] → screen
// Frame 2: [JS] [rAF fires] [Layout] [Paint] → screen
//          ↑ GOOD: rAF fires right before layout — maximum preparation time

// The rAF callback has the MAXIMUM time available before the next paint
// setTimeout fires mid-frame — leaves less time for the rest of the pipeline
```

---

## 5. Frame Budgeting with rAF

At 60fps, each frame has 16.67ms. Your rAF callback must complete within this budget.

### Measuring Time Budget Usage

```javascript
function createFrameMonitor(warnThresholdMs = 10) {
  return function monitoredRAF(callback) {
    return requestAnimationFrame((timestamp) => {
      const start = performance.now();
      callback(timestamp);
      const elapsed = performance.now() - start;

      if (elapsed > warnThresholdMs) {
        console.warn(
          `rAF callback took ${elapsed.toFixed(1)}ms`,
          `(budget: 16.67ms, used: ${((elapsed / 16.67) * 100).toFixed(0)}%)`,
        );
      }
    });
  };
}

const raf = createFrameMonitor(10);

function animate(timestamp) {
  // Your animation code here
  raf(animate);
}
raf(animate);
```

### Splitting Heavy Work Across Frames

```javascript
// ❌ Too much work in one rAF callback — drops frames
function animate(timestamp) {
  for (let i = 0; i < 10_000; i++) {
    heavyCalculation(i); // 50ms total — drops 3 frames!
  }
  requestAnimationFrame(animate);
}

// ✅ Chunk work — do some per frame
class ChunkedAnimator {
  constructor(totalWork) {
    this._queue = Array.from({ length: totalWork }, (_, i) => i);
    this._chunkSize = 100; // process 100 items per frame
    this._rafId = null;
  }

  start() {
    this._rafId = requestAnimationFrame((t) => this._frame(t));
  }

  _frame(timestamp) {
    const budgetMs = 8; // use only 8ms of the 16ms budget
    const deadline = timestamp + budgetMs;

    while (this._queue.length > 0 && performance.now() < deadline) {
      heavyCalculation(this._queue.shift());
    }

    if (this._queue.length > 0) {
      this._rafId = requestAnimationFrame((t) => this._frame(t));
    } else {
      console.log("Work complete");
    }
  }

  stop() {
    if (this._rafId !== null) {
      cancelAnimationFrame(this._rafId);
      this._rafId = null;
    }
  }
}
```

---

## 6. Cancelling rAF — Memory and CPU Safety

Always cancel rAF callbacks when they're no longer needed.

```javascript
class Component {
  constructor(element) {
    this._element = element;
    this._rafId = null;
    this._running = false;
  }

  startAnimation() {
    if (this._running) return;
    this._running = true;
    this._rafId = requestAnimationFrame((t) => this._loop(t));
  }

  stopAnimation() {
    this._running = false;
    if (this._rafId !== null) {
      cancelAnimationFrame(this._rafId);
      this._rafId = null;
    }
  }

  _loop(timestamp) {
    if (!this._running) return; // double-check in case stop() was called

    this._update(timestamp);
    this._rafId = requestAnimationFrame((t) => this._loop(t));
  }

  _update(timestamp) {
    /* animation logic */
  }

  // Must be called when component is removed from DOM
  destroy() {
    this.stopAnimation();
    this._element = null;
  }
}

// ❌ Common leak: rAF continues after element is gone
function startForever() {
  function loop() {
    element.style.transform = "..."; // element may be removed — this throws or is wasted
    requestAnimationFrame(loop); // loops forever even if element is gone
  }
  requestAnimationFrame(loop);
}
```

### Page Visibility — Auto-Pause

```javascript
// ✅ Automatically pause when tab is hidden
document.addEventListener("visibilitychange", () => {
  if (document.hidden) {
    animator.stop(); // save CPU and battery
  } else {
    animator.start(); // resume when tab is visible
  }
});
```

---

## 7. Multiple Animations — One Loop vs Many

### ❌ Many separate rAF loops

```javascript
// Each animation has its own rAF — wasteful
startParticleAnimation(); // → requestAnimationFrame(particleLoop)
startScrollIndicator(); // → requestAnimationFrame(scrollLoop)
startCounterAnimation(); // → requestAnimationFrame(counterLoop)
startBackgroundEffect(); // → requestAnimationFrame(bgLoop)

// Problem: 4 rAF callbacks per frame
// Each callback has overhead
// Browser can't optimize them together
```

### ✅ Single animation loop — subscriber pattern

```javascript
class AnimationScheduler {
  #subscribers = new Set(); // { id, update } callbacks
  #rafId = null;
  #timestamp = 0;

  add(id, updateFn) {
    this.#subscribers.add({ id, update: updateFn });
    if (!this.#rafId) {
      this.#rafId = requestAnimationFrame((t) => this.#loop(t));
    }
    return () => this.remove(id); // return unsubscribe fn
  }

  remove(id) {
    for (const sub of this.#subscribers) {
      if (sub.id === id) {
        this.#subscribers.delete(sub);
        break;
      }
    }
    if (this.#subscribers.size === 0) {
      cancelAnimationFrame(this.#rafId);
      this.#rafId = null;
    }
  }

  #loop(timestamp) {
    this.#timestamp = timestamp;

    for (const { update } of this.#subscribers) {
      try {
        update(timestamp);
      } catch (err) {
        console.error("Animation subscriber error:", err);
      }
    }

    if (this.#subscribers.size > 0) {
      this.#rafId = requestAnimationFrame((t) => this.#loop(t));
    } else {
      this.#rafId = null;
    }
  }
}

// Global scheduler — one rAF for all animations
const scheduler = new AnimationScheduler();

// Add subscribers
const unsubParticles = scheduler.add("particles", (t) => updateParticles(t));
const unsubScroll = scheduler.add("scroll", (t) => updateScrollIndicator(t));
const unsubCounter = scheduler.add("counter", (t) => updateCounter(t));

// Remove when done
unsubParticles(); // removes only particles subscriber
// If no subscribers remain: rAF loop stops automatically
```

---

## 8. rAF and Layout Reads — The Correct Pattern

The relationship between rAF and layout reads is subtle and important.

### When to Read Layout Properties

```javascript
// rAF fires BEFORE paint, but AFTER layout from the previous frame
// Layout properties are accurate at the START of a rAF callback

function animate(timestamp) {
  // ✅ Reading layout at START of rAF is safe — values are fresh
  const rect = element.getBoundingClientRect(); // layout is current

  // ✅ Writing after reading — no thrashing
  element.style.transform = `translateX(${computeNewX(rect)}px)`;

  requestAnimationFrame(animate);
}
```

### The Double rAF Pattern

Sometimes you need to read a value AFTER a write has been applied. Use double rAF:

```javascript
// Single rAF: executes BEFORE paint
// Double rAF: executes AFTER paint (one frame later)

requestAnimationFrame(() => {
  // This runs before Frame N's paint
  element.style.display = "block";
  element.style.transition = "opacity 0.3s";
  // Element is in DOM but browser hasn't painted yet

  requestAnimationFrame(() => {
    // This runs before Frame N+1's paint
    // The display:block has now been applied and laid out
    element.style.opacity = "1"; // transition starts correctly
    // Without double rAF: transition might not trigger
    // (browser might batch display + opacity as one change)
  });
});
```

---

## 9. rAF Throttling for Non-Visual Work

rAF fires before every paint — too often for work that doesn't need per-frame execution.

```javascript
// ❌ Updating analytics on every frame — wasteful
function trackScrollDepth() {
  analytics.track("scroll", { depth: window.scrollY });
  requestAnimationFrame(trackScrollDepth); // 60 calls/second!
}

// ✅ Throttle to a reasonable rate
class RAFThrottler {
  constructor(callback, maxFPS = 10) {
    this._callback = callback;
    this._interval = 1000 / maxFPS; // ms between calls
    this._lastTime = 0;
    this._rafId = null;
  }

  start() {
    this._rafId = requestAnimationFrame((t) => this._tick(t));
  }

  _tick(timestamp) {
    this._rafId = requestAnimationFrame((t) => this._tick(t));

    if (timestamp - this._lastTime >= this._interval) {
      this._lastTime = timestamp;
      this._callback(timestamp);
    }
  }

  stop() {
    cancelAnimationFrame(this._rafId);
  }
}

const scrollTracker = new RAFThrottler(() => {
  analytics.track("scroll", { depth: window.scrollY });
}, 5); // max 5 times per second

scrollTracker.start();
```

---

## 10. requestIdleCallback — The Complement

`requestIdleCallback` (rIC) is rAF's complement for non-urgent work. It fires during idle time — after rendering is complete and before the next frame.

```
Frame budget used:
  [Input] [JS rAF] [Layout] [Paint] [Composite] [idle time] → next frame

rAF: fires at start of rendering (right timing for visual work)
rIC: fires during idle time (right timing for background work)
```

```javascript
// ✅ Non-urgent work in rIC — doesn't affect animation
function runBackgroundWork() {
  requestIdleCallback(
    (deadline) => {
      // deadline.timeRemaining(): ms left before browser needs to render
      // deadline.didTimeout: true if called due to timeout, not idle time

      while (deadline.timeRemaining() > 5 && workQueue.length > 0) {
        processNextItem(workQueue.shift());
      }

      if (workQueue.length > 0) {
        requestIdleCallback(runBackgroundWork);
      }
    },
    { timeout: 2000 },
  ); // force run within 2 seconds even if not idle
}

// Good for: analytics, prefetching, cleanup, non-visible updates
// Bad for: visual animations, user-response UI updates
```

### rAF + rIC Pattern — Visual + Background

```javascript
// Pattern: visual updates in rAF, background work in rIC

const workQueue = [];
let rafScheduled = false;
let ricScheduled = false;

// Schedule visual update
function scheduleRender() {
  if (!rafScheduled) {
    rafScheduled = true;
    requestAnimationFrame(() => {
      rafScheduled = false;
      renderVisibleContent(); // only renders what's needed
    });
  }
}

// Schedule background work
function scheduleBackground(task) {
  workQueue.push(task);
  if (!ricScheduled) {
    ricScheduled = true;
    requestIdleCallback((deadline) => {
      ricScheduled = false;
      while (deadline.timeRemaining() > 1 && workQueue.length > 0) {
        workQueue.shift()();
      }
      if (workQueue.length > 0) scheduleBackground(); // still more work
    });
  }
}
```

---

## 11. rAF in Web Workers

`requestAnimationFrame` is not available in Web Workers (they have no DOM access). But with `OffscreenCanvas`, you can animate on the GPU from a Worker:

```javascript
// main.js
const canvas = document.getElementById("canvas");
const offscreen = canvas.transferControlToOffscreen();
const worker = new Worker("./animation-worker.js", { type: "module" });

worker.postMessage({ canvas: offscreen }, [offscreen]);
```

```javascript
// animation-worker.js
let ctx;
let lastTime = null;

self.onmessage = ({ data }) => {
  ctx = data.canvas.getContext("2d");
  requestAnimationFrame(draw); // rAF IS available in OffscreenCanvas workers
};

function draw(timestamp) {
  const delta = lastTime ? (timestamp - lastTime) / 1000 : 0;
  lastTime = timestamp;

  ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
  // ... draw complex scenes entirely on worker thread
  // Main thread is completely free

  requestAnimationFrame(draw);
}
```

---

## 12. Debugging Animation Performance

### Chrome DevTools — Performance Tab

```
Key things to look for in animation frames:

1. Frame timeline at top:
   Green = frames meeting budget (< 16ms)
   Yellow = frames slightly over budget (16-33ms)
   Red = dropped frames (> 33ms)

2. "Rendering" row in timeline:
   Shows: Style, Layout, Paint, Composite per frame
   Purple blocks = layout/paint (expensive!)
   Should be minimal for transform/opacity animations

3. Main thread flame graph:
   "Animation Frame Fired" entries = your rAF callbacks
   Width = execution time
   Should be < 10ms per frame for headroom

4. CPU usage:
   Smooth animation: low, consistent CPU
   Jank: CPU spikes corresponding to dropped frames
```

### Measuring Animation Frame Rate

```javascript
class FPSMonitor {
  constructor() {
    this._frames = [];
    this._running = false;
    this._rafId = null;
  }

  start() {
    this._frames = [];
    this._running = true;
    this._rafId = requestAnimationFrame((t) => this._frame(t));
  }

  stop() {
    this._running = false;
    cancelAnimationFrame(this._rafId);
  }

  _frame(timestamp) {
    this._frames.push(timestamp);

    // Keep only last 60 frames
    if (this._frames.length > 60) this._frames.shift();

    if (this._running) {
      this._rafId = requestAnimationFrame((t) => this._frame(t));
    }
  }

  get fps() {
    if (this._frames.length < 2) return 0;
    const elapsed = this._frames[this._frames.length - 1] - this._frames[0];
    return Math.round((this._frames.length - 1) / (elapsed / 1000));
  }

  get frameDrops() {
    let drops = 0;
    for (let i = 1; i < this._frames.length; i++) {
      const delta = this._frames[i] - this._frames[i - 1];
      if (delta > 20) drops++; // > 20ms gap = dropped frame (budget is 16.67ms)
    }
    return drops;
  }
}

const monitor = new FPSMonitor();
monitor.start();

// Overlay FPS in corner
setInterval(() => {
  document.getElementById("fps").textContent =
    `${monitor.fps} FPS (${monitor.frameDrops} drops)`;
}, 500);
```

---

## 13. Common Animation Patterns

### Spring Animation

```javascript
class SpringAnimator {
  constructor({ stiffness = 180, damping = 12 } = {}) {
    this._k = stiffness; // spring stiffness
    this._d = damping; // damping coefficient
  }

  createSpring(initial = 0) {
    return {
      current: initial,
      target: initial,
      velocity: 0,
    };
  }

  step(spring, deltaSeconds) {
    const force = -this._k * (spring.current - spring.target);
    const damping = -this._d * spring.velocity;
    const acceleration = force + damping;

    spring.velocity += acceleration * deltaSeconds;
    spring.current += spring.velocity * deltaSeconds;

    // Snap when close enough
    const isSettled =
      Math.abs(spring.velocity) < 0.01 &&
      Math.abs(spring.current - spring.target) < 0.01;
    if (isSettled) {
      spring.current = spring.target;
      spring.velocity = 0;
    }

    return isSettled;
  }
}

// Usage
const spring = new SpringAnimator({ stiffness: 200, damping: 20 });
const xSpring = spring.createSpring(0);
let settled = false;

document.getElementById("btn").addEventListener("click", () => {
  xSpring.target = 200; // set new target
  settled = false;
  if (!animating) startAnimation();
});

function animate(timestamp) {
  const delta = lastTimestamp ? (timestamp - lastTimestamp) / 1000 : 0;
  lastTimestamp = timestamp;

  settled = spring.step(xSpring, delta);
  element.style.transform = `translateX(${xSpring.current}px)`;

  if (!settled) {
    requestAnimationFrame(animate);
    animating = true;
  } else {
    animating = false;
  }
}
```

### FLIP Animation (First-Last-Invert-Play)

```javascript
// FLIP: animate between two DOM positions using transform
// Avoids layout-triggering animations

function flipAnimation(element, newParent) {
  // First: record initial position
  const first = element.getBoundingClientRect();

  // (DOM change that triggers layout)
  newParent.appendChild(element);

  // Last: record final position
  const last = element.getBoundingClientRect();

  // Invert: move element back to initial position using transform
  const deltaX = first.left - last.left;
  const deltaY = first.top - last.top;

  element.style.transform = `translate(${deltaX}px, ${deltaY}px)`;
  element.style.transition = "none"; // no transition while inverting

  // Play: remove the invert transform — CSS transition plays
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      // Double rAF: ensure layout sees the invert position before transition
      element.style.transition = "transform 300ms ease";
      element.style.transform = ""; // animate to natural position
    });
  });
}
```

---

## 14. Good Practices

### ✅ Always use delta time for physics/velocity

```javascript
// ✅ Velocity in units per second, multiplied by delta
(velocity * (timestamp - lastTimestamp)) / 1000;
```

### ✅ Cap delta to prevent large jumps on tab focus

```javascript
// ✅ Prevents animation from jumping after tab is hidden
const delta = Math.min((timestamp - lastTime) / 1000, 0.1);
```

### ✅ Cancel rAF in cleanup/destroy

```javascript
// ✅ Always cancel on unmount/destroy
destroy() {
  cancelAnimationFrame(this._rafId);
}
```

### ✅ One shared rAF loop for all animations

```javascript
// ✅ Single scheduler handles all animations efficiently
scheduler.add("myAnimation", update);
// vs multiple independent rAF loops
```

### ✅ Use CSS animations for simple transitions — rAF only for complex logic

```css
/* ✅ CSS handles simple transitions — no rAF needed */
.card {
  transition:
    transform 300ms ease,
    opacity 300ms ease;
}
.card:hover {
  transform: translateY(-4px);
}
```

```javascript
// rAF for: physics, spring, particle systems, game-style animation
// CSS for: hover effects, enter/exit transitions, simple state changes
```

---

## 15. Bad Practices

### ❌ Using setTimeout for animations

```javascript
// ❌ Not synced to display — causes tearing and inconsistent timing
function animate() {
  x += 3;
  element.style.transform = `translateX(${x}px)`;
  setTimeout(animate, 16);
}
```

### ❌ Multiple rAF loops for the same component

```javascript
// ❌ Three rAF loops in one component
requestAnimationFrame(updatePosition);
requestAnimationFrame(updateColor);
requestAnimationFrame(updateOpacity);
// All three run every frame — consolidate into one
```

### ❌ Doing heavy work every frame

```javascript
// ❌ Sorting 1000 items every frame
function animate() {
  const sorted = data.sort((a, b) => a.value - b.value); // expensive!
  render(sorted);
  requestAnimationFrame(animate);
}

// ✅ Sort once, re-sort only when data changes
```

### ❌ Not checking if animation is still needed

```javascript
// ❌ Continues animating even when finished
function animate() {
  x = Math.lerp(x, target, 0.1);
  element.style.transform = `translateX(${x}px)`;
  requestAnimationFrame(animate); // runs forever, even when x ≈ target
}

// ✅ Stop when close enough
function animate() {
  x = lerp(x, target, 0.1);
  element.style.transform = `translateX(${x}px)`;
  if (Math.abs(x - target) > 0.1) {
    requestAnimationFrame(animate);
  }
  // else: animation finished, loop stops
}
```

---

## 16. Common Mistakes

### Mistake 1 — Stale closure in animation loop

```javascript
// ❌ target captured at closure creation — never updates
const target = 100;
function animate() {
  x = lerp(x, target, 0.1); // target is always 100
  // Even if user clicks to change target, this closure sees old value
  requestAnimationFrame(animate);
}

// ✅ Use a ref or module variable
let target = 100;
function animate() {
  x = lerp(x, target, 0.1); // reads latest value
  requestAnimationFrame(animate);
}
// target can be updated from anywhere
```

### Mistake 2 — Accumulating rAF callbacks

```javascript
// ❌ Each event schedules a new rAF, but old ones never cancel
window.addEventListener("mousemove", (e) => {
  requestAnimationFrame(() => updateCursor(e.clientX, e.clientY));
  // 1000 mousemove events = 1000 pending rAF callbacks!
});

// ✅ Deduplicate: only one pending callback at a time
let pendingRAF = null;
let mouseX = 0,
  mouseY = 0;

window.addEventListener("mousemove", (e) => {
  mouseX = e.clientX;
  mouseY = e.clientY;
  if (!pendingRAF) {
    pendingRAF = requestAnimationFrame(() => {
      pendingRAF = null;
      updateCursor(mouseX, mouseY); // uses latest values
    });
  }
});
```

### Mistake 3 — Not understanding rAF timing relative to scroll events

```javascript
// scroll fires → event handler runs → (possibly) rAF fires → layout → paint
// rAF happens AFTER scroll handling, BEFORE layout

window.addEventListener("scroll", () => {
  // Don't schedule rAF here for scroll-based animations
  // Instead: update position IMMEDIATELY in scroll handler
  // Or: use CSS scroll-driven animations (compositor thread)
  element.style.transform = `translateY(${-window.scrollY * 0.5}px)`;
  // This is fine — change is batched, applied before next layout
});
```

---

## 17. Interview-Level Explanation

> **"What is requestAnimationFrame? How does it differ from setTimeout? When should you use each?"**

**Strong answer:**

> "requestAnimationFrame is a browser API that schedules a callback to run before the next paint, synchronized to the display's refresh rate. It's the correct tool for all visual animations because it guarantees your changes are included in the current frame — the browser will apply your DOM mutations and then immediately paint.
>
> The key differences from setTimeout: rAF is synced to the display's vsync (exactly 60fps on a 60Hz display, 120fps on a 120Hz display), while setTimeout fires at approximate intervals that aren't synchronized to painting. rAF fires before layout and paint in each frame, while setTimeout fires at arbitrary points in the event loop. rAF is automatically paused in background tabs to save battery, while setTimeout continues running. And rAF provides a precise high-resolution timestamp that you use to calculate time-based velocity, making animations frame-rate independent.
>
> The correct animation pattern uses the timestamp to calculate a delta — the time elapsed since the last frame in seconds — then multiplies velocity by delta. This way an animation that moves 200px per second moves that distance whether the display is 30fps or 120fps.
>
> You should always cancel the rAF handle in cleanup — if the component is removed from the DOM, the animation loop should stop. Forgetting this is a common memory and CPU leak. For components with multiple visual things to animate, a single shared scheduler with a subscriber pattern is more efficient than multiple independent rAF loops.
>
> For non-visual background work — analytics, prefetching, cleanup — use requestIdleCallback instead. It runs during the idle time after rendering is complete, so it doesn't compete with the frame budget."

---

## 18. Exercises

### Exercise 1 — Fix the animation timing

```javascript
// ❌ This animation runs differently on 60fps vs 120fps monitors
// Fix it to be frame-rate independent

let x = 0;

function animate() {
  x += 5; // always 5px per frame — different speed at different framerates
  element.style.transform = `translateX(${x}px)`;
  if (x < 300) requestAnimationFrame(animate);
}

requestAnimationFrame(animate);
```

<details>
<summary>Solution</summary>

```javascript
// ✅ Frame-rate independent using delta time
const SPEED = 300; // 300 pixels per second
let x = 0;
let lastTime = null;

function animate(timestamp) {
  if (lastTime === null) {
    lastTime = timestamp;
    requestAnimationFrame(animate);
    return;
  }

  const delta = Math.min((timestamp - lastTime) / 1000, 0.1); // cap at 100ms
  lastTime = timestamp;

  x += SPEED * delta;
  element.style.transform = `translateX(${x}px)`;

  if (x < 300) {
    requestAnimationFrame(animate);
  } else {
    element.style.transform = `translateX(300px)`; // snap to final
  }
}

requestAnimationFrame(animate);

// At 60fps: ~5px per frame (300 × 1/60 = 5)
// At 120fps: ~2.5px per frame (300 × 1/120 = 2.5)
// Total time to cover 300px: always exactly 1 second
```

</details>

---

### Exercise 2 — Build an animation scheduler

Build a scheduler that:

- Accepts subscriber functions with an id
- Runs all subscribers in a single rAF loop
- Stops the loop when no subscribers remain
- Returns an unsubscribe function from `add()`

```javascript
const scheduler = createAnimationScheduler();

const unsub1 = scheduler.add("particles", (timestamp) =>
  updateParticles(timestamp),
);
const unsub2 = scheduler.add("spinner", (timestamp) =>
  updateSpinner(timestamp),
);

unsub1(); // removes only particles
// Loop continues for spinner
// When unsub2() is called: loop stops entirely
```

<details>
<summary>Solution</summary>

```javascript
function createAnimationScheduler() {
  const subscribers = new Map(); // id → callback
  let rafId = null;

  function loop(timestamp) {
    for (const [id, callback] of subscribers) {
      try {
        callback(timestamp);
      } catch (err) {
        console.error(`Animation subscriber "${id}" threw:`, err);
      }
    }

    if (subscribers.size > 0) {
      rafId = requestAnimationFrame(loop);
    } else {
      rafId = null;
    }
  }

  return {
    add(id, callback) {
      subscribers.set(id, callback);

      if (rafId === null) {
        rafId = requestAnimationFrame(loop);
      }

      return function unsubscribe() {
        subscribers.delete(id);
        // Loop will stop naturally after current frame if empty
      };
    },

    has(id) {
      return subscribers.has(id);
    },

    get size() {
      return subscribers.size;
    },
  };
}
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md) — Where rAF fits in the rendering pipeline
- [`javascript-core/04-microtask-vs-macrotask.md`](../javascript-core/04-microtask-vs-macrotask.md) — rAF in the event loop
- [`performance/10-canvas-optimization.md`](./10-canvas-optimization.md) — rAF for canvas animation
- [`animations/03-compositor-animations.md`](../animations/03-compositor-animations.md) — CSS vs rAF animations
- [`rendering/03-cooperative-scheduling.md`](../rendering/03-cooperative-scheduling.md) — Splitting work across frames

---

<div align="center">

**Next:** [`performance/05-memory-leaks.md`](./05-memory-leaks.md) →

</div>
