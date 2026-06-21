# 02 — JavaScript Animations

> **"CSS animations describe states. JavaScript animations describe behavior. The moment your animation needs to respond to physics, user input velocity, or runtime-computed values, you've left the world CSS was designed for — and that's exactly when requestAnimationFrame and the Web Animations API earn their keep."**

CSS handles the majority of UI animation needs, but some scenarios demand JavaScript: physics-based motion, animations driven by gesture velocity, animations that need to be paused/reversed/scrubbed programmatically, or animations synchronized with non-CSS state. This document covers requestAnimationFrame fundamentals, the Web Animations API, spring physics, gesture-driven animation, and the libraries that make complex JavaScript animation tractable.

---

## 📚 Table of Contents

1. [requestAnimationFrame Fundamentals](#1-requestanimationframe-fundamentals)
2. [The Web Animations API](#2-the-web-animations-api)
3. [Animating with rAF Manually](#3-animating-with-raf-manually)
4. [Easing Functions in JavaScript](#4-easing-functions-in-javascript)
5. [Spring Physics Animation](#5-spring-physics-animation)
6. [Gesture-Driven Animation](#6-gesture-driven-animation)
7. [Animation Libraries](#7-animation-libraries)
8. [Framer Motion Patterns](#8-framer-motion-patterns)
9. [Scroll-Driven Animation](#9-scroll-driven-animation)
10. [Canvas and WebGL Animation](#10-canvas-and-webgl-animation)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. requestAnimationFrame Fundamentals

```javascript
// Basic rAF loop
function animate(timestamp) {
  // timestamp: DOMHighResTimeStamp, same clock as performance.now()
  updateAnimationState(timestamp);
  requestAnimationFrame(animate); // schedule next frame
}
requestAnimationFrame(animate);

// rAF guarantees:
// - Called before the next repaint (synced to display refresh rate, typically 60Hz)
// - Paused automatically when tab is not visible (saves battery/CPU)
// - Higher precision than setTimeout/setInterval for visual updates

// Calculating delta time for frame-rate-independent animation
let lastTime = null;

function animate(timestamp) {
  if (lastTime === null) lastTime = timestamp;
  const deltaMs = timestamp - lastTime;
  lastTime = timestamp;

  updatePosition(deltaMs); // movement scaled by actual elapsed time
  requestAnimationFrame(animate);
}

function updatePosition(deltaMs) {
  const speedPxPerMs = 0.3;
  position += speedPxPerMs * deltaMs;
  // On 60Hz: deltaMs ≈ 16.7ms per frame
  // On 144Hz: deltaMs ≈ 6.9ms per frame
  // Using deltaMs ensures consistent VISUAL speed regardless of refresh rate
}
```

### Canceling Animation Frames

```javascript
class AnimationLoop {
  #rafId = null;
  #running = false;

  start(callback) {
    this.#running = true;
    const loop = (timestamp) => {
      if (!this.#running) return;
      callback(timestamp);
      this.#rafId = requestAnimationFrame(loop);
    };
    this.#rafId = requestAnimationFrame(loop);
  }

  stop() {
    this.#running = false;
    if (this.#rafId) cancelAnimationFrame(this.#rafId);
  }
}

// React cleanup pattern
useEffect(() => {
  let rafId;
  function tick() {
    updateAnimation();
    rafId = requestAnimationFrame(tick);
  }
  rafId = requestAnimationFrame(tick);
  return () => cancelAnimationFrame(rafId); // ALWAYS clean up
}, []);
```

---

## 2. The Web Animations API

The Web Animations API (WAAPI) provides JavaScript control over animations with native browser performance — it runs on the compositor when possible, just like CSS animations.

```javascript
// Basic element.animate()
const animation = element.animate(
  [
    { transform: "translateY(0)", opacity: 1 }, // keyframe 1
    { transform: "translateY(-20px)", opacity: 0 }, // keyframe 2
  ],
  {
    duration: 300,
    easing: "cubic-bezier(0.4, 0, 0.2, 1)",
    fill: "forwards",
  },
);

// Returns an Animation object with full programmatic control:
animation.pause();
animation.play();
animation.reverse();
animation.cancel(); // removes all effects, reverts to original style
animation.finish(); // jumps to end state immediately
animation.currentTime = 150; // scrub to specific time (ms)
animation.playbackRate = 2; // play at 2x speed
animation.playbackRate = -1; // play in reverse

// Promise-based completion
await animation.finished;
console.log("Animation complete");

// Event-based
animation.addEventListener("finish", () => console.log("done"));
animation.addEventListener("cancel", () => console.log("cancelled"));
```

### Keyframe Formats

```javascript
// Array of keyframe objects (offset is implicit: evenly spaced)
element.animate([{ opacity: 0 }, { opacity: 0.5 }, { opacity: 1 }], {
  duration: 600,
});

// Explicit offsets (0 to 1)
element.animate(
  [
    { transform: "scale(1)", offset: 0 },
    { transform: "scale(1.2)", offset: 0.3 }, // reaches peak at 30% through
    { transform: "scale(1)", offset: 1 },
  ],
  { duration: 600 },
);

// Object format (property: array of values)
element.animate(
  {
    opacity: [0, 1],
    transform: ["translateY(20px)", "translateY(0)"],
  },
  { duration: 300, easing: "ease-out" },
);

// Per-keyframe easing
element.animate(
  [
    { transform: "translateX(0)", easing: "ease-in" },
    { transform: "translateX(100px)", easing: "ease-out" },
    { transform: "translateX(50px)" },
  ],
  { duration: 600 },
);
```

### Animation Options

```javascript
element.animate(keyframes, {
  duration: 300,
  delay: 0,
  endDelay: 0,
  easing: "ease-out",
  iterations: Infinity, // or a number
  direction: "alternate", // 'normal' | 'reverse' | 'alternate' | 'alternate-reverse'
  fill: "forwards", // 'none' | 'forwards' | 'backwards' | 'both'
  iterationStart: 0, // start partway through an iteration (0-1)
  composite: "replace", // 'replace' | 'add' | 'accumulate'
});
```

### Animating Multiple Elements with Shared Timeline

```javascript
// Group animations using a shared timing reference
const stagger = (elements, keyframes, baseOptions) => {
  return elements.map((el, i) =>
    el.animate(keyframes, {
      ...baseOptions,
      delay: i * 50, // staggered start
    }),
  );
};

const animations = stagger(
  [...document.querySelectorAll(".list-item")],
  [
    { opacity: 0, transform: "translateY(10px)" },
    { opacity: 1, transform: "translateY(0)" },
  ],
  { duration: 300, easing: "ease-out", fill: "forwards" },
);

// Wait for ALL to complete
await Promise.all(animations.map((a) => a.finished));
```

---

## 3. Animating with rAF Manually

For animations that need custom logic per frame (not expressible as keyframes):

```javascript
function animateValue({
  from,
  to,
  duration,
  easing = (t) => t,
  onUpdate,
  onComplete,
}) {
  const startTime = performance.now();

  function tick(now) {
    const elapsed = now - startTime;
    const progress = Math.min(elapsed / duration, 1);
    const eased = easing(progress);
    const value = from + (to - from) * eased;

    onUpdate(value);

    if (progress < 1) {
      requestAnimationFrame(tick);
    } else {
      onComplete?.();
    }
  }

  requestAnimationFrame(tick);
}

// Usage: animate a counter
animateValue({
  from: 0,
  to: 1000,
  duration: 1500,
  easing: (t) => 1 - Math.pow(1 - t, 3), // ease-out cubic
  onUpdate: (value) => {
    counterElement.textContent = Math.round(value).toLocaleString();
  },
  onComplete: () => console.log("Counter animation done"),
});
```

### Cancellable Animation Class

```typescript
class TweenAnimation {
  #rafId: number | null = null;
  #startTime: number | null = null;
  #cancelled = false;

  constructor(
    private from: number,
    private to: number,
    private duration: number,
    private easing: (t: number) => number,
    private onUpdate: (value: number) => void,
  ) {}

  start(): Promise<void> {
    return new Promise((resolve) => {
      const tick = (now: number) => {
        if (this.#cancelled) return resolve();
        if (this.#startTime === null) this.#startTime = now;

        const elapsed = now - this.#startTime;
        const progress = Math.min(elapsed / this.duration, 1);
        const value = this.from + (this.to - this.from) * this.easing(progress);

        this.onUpdate(value);

        if (progress < 1) {
          this.#rafId = requestAnimationFrame(tick);
        } else {
          resolve();
        }
      };
      this.#rafId = requestAnimationFrame(tick);
    });
  }

  cancel(): void {
    this.#cancelled = true;
    if (this.#rafId) cancelAnimationFrame(this.#rafId);
  }
}
```

---

## 4. Easing Functions in JavaScript

```javascript
// Penner's easing equations (simplified, normalized 0-1 input/output)
const Easing = {
  linear: (t) => t,

  easeInQuad: (t) => t * t,
  easeOutQuad: (t) => t * (2 - t),
  easeInOutQuad: (t) => (t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t),

  easeInCubic: (t) => t * t * t,
  easeOutCubic: (t) => 1 - Math.pow(1 - t, 3),
  easeInOutCubic: (t) =>
    t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2,

  easeOutElastic: (t) => {
    const c4 = (2 * Math.PI) / 3;
    return t === 0
      ? 0
      : t === 1
        ? 1
        : Math.pow(2, -10 * t) * Math.sin((t * 10 - 0.75) * c4) + 1;
  },

  easeOutBounce: (t) => {
    const n1 = 7.5625,
      d1 = 2.75;
    if (t < 1 / d1) return n1 * t * t;
    if (t < 2 / d1) return n1 * (t -= 1.5 / d1) * t + 0.75;
    if (t < 2.5 / d1) return n1 * (t -= 2.25 / d1) * t + 0.9375;
    return n1 * (t -= 2.625 / d1) * t + 0.984375;
  },

  // cubic-bezier implementation matching CSS curves
  cubicBezier: (x1, y1, x2, y2) => (t) => {
    // Newton-Raphson iteration to solve for t given x progress
    const cx = 3 * x1,
      bx = 3 * (x2 - x1) - cx,
      ax = 1 - cx - bx;
    const cy = 3 * y1,
      by = 3 * (y2 - y1) - cy,
      ay = 1 - cy - by;

    const sampleX = (t) => ((ax * t + bx) * t + cx) * t;
    const sampleY = (t) => ((ay * t + by) * t + cy) * t;

    let x = t;
    for (let i = 0; i < 8; i++) {
      const currentX = sampleX(x) - t;
      if (Math.abs(currentX) < 1e-6) break;
      const derivative = (3 * ax * x + 2 * bx) * x + cx;
      x -= currentX / derivative;
    }
    return sampleY(x);
  },
};

// Usage: same curve as CSS's "ease-out"
const cssEaseOut = Easing.cubicBezier(0, 0, 0.58, 1);
```

---

## 5. Spring Physics Animation

Spring-based animation produces more natural, organic motion than time-based easing curves, especially for interruptible/gesture-driven UI.

```javascript
// Simple spring physics simulation
class Spring {
  #position;
  #velocity = 0;
  #target;
  #stiffness;
  #damping;
  #mass;

  constructor({ from, to, stiffness = 170, damping = 26, mass = 1 }) {
    this.#position = from;
    this.#target = to;
    this.#stiffness = stiffness;
    this.#damping = damping;
    this.#mass = mass;
  }

  // Advance simulation by deltaMs, return current position
  update(deltaMs) {
    const dt = Math.min(deltaMs / 1000, 0.064); // clamp for stability

    const springForce = -this.#stiffness * (this.#position - this.#target);
    const dampingForce = -this.#damping * this.#velocity;
    const acceleration = (springForce + dampingForce) / this.#mass;

    this.#velocity += acceleration * dt;
    this.#position += this.#velocity * dt;

    return this.#position;
  }

  isSettled(threshold = 0.01) {
    return (
      Math.abs(this.#target - this.#position) < threshold &&
      Math.abs(this.#velocity) < threshold
    );
  }

  setTarget(target) {
    this.#target = target; // can change mid-animation — spring naturally adjusts
  }
}

// Usage: drag-to-dismiss with spring snap-back
function useSpringDrag(elementRef) {
  const springRef = useRef(null);
  const rafRef = useRef(null);

  function startDrag(initialY) {
    springRef.current = new Spring({
      from: initialY,
      to: 0,
      stiffness: 300,
      damping: 30,
    });
  }

  function onDragMove(currentY) {
    springRef.current?.setTarget(currentY); // spring chases the finger
  }

  function onDragEnd() {
    springRef.current?.setTarget(0); // spring back to rest position

    let lastTime = performance.now();
    function tick(now) {
      const delta = now - lastTime;
      lastTime = now;

      const pos = springRef.current.update(delta);
      elementRef.current.style.transform = `translateY(${pos}px)`;

      if (!springRef.current.isSettled()) {
        rafRef.current = requestAnimationFrame(tick);
      }
    }
    rafRef.current = requestAnimationFrame(tick);
  }

  return { startDrag, onDragMove, onDragEnd };
}
```

---

## 6. Gesture-Driven Animation

```javascript
// Drag-to-dismiss card with velocity-based release animation
function useDragToDismiss(onDismiss) {
  const elementRef = useRef(null);
  const stateRef = useRef({
    startY: 0,
    currentY: 0,
    lastY: 0,
    lastTime: 0,
    velocity: 0,
  });

  useEffect(() => {
    const el = elementRef.current;
    if (!el) return;

    function onPointerDown(e) {
      stateRef.current.startY = e.clientY;
      stateRef.current.lastY = e.clientY;
      stateRef.current.lastTime = performance.now();
      el.style.transition = "none"; // disable CSS transition during drag
      el.setPointerCapture(e.pointerId);
    }

    function onPointerMove(e) {
      const deltaY = e.clientY - stateRef.current.startY;
      const now = performance.now();
      const dt = now - stateRef.current.lastTime;

      // Track velocity (px/ms) for momentum on release
      stateRef.current.velocity =
        (e.clientY - stateRef.current.lastY) / Math.max(dt, 1);
      stateRef.current.lastY = e.clientY;
      stateRef.current.lastTime = now;

      if (deltaY > 0) {
        // only allow downward drag
        el.style.transform = `translateY(${deltaY}px)`;
        el.style.opacity = String(1 - deltaY / 300);
      }
    }

    function onPointerUp(e) {
      const deltaY = e.clientY - stateRef.current.startY;
      const velocity = stateRef.current.velocity;

      el.style.transition = "transform 0.3s ease-out, opacity 0.3s ease-out";

      // Dismiss if dragged far enough OR flicked with high velocity
      const shouldDismiss = deltaY > 100 || velocity > 0.5;

      if (shouldDismiss) {
        el.style.transform = `translateY(${window.innerHeight}px)`;
        el.style.opacity = "0";
        el.addEventListener("transitionend", () => onDismiss(), { once: true });
      } else {
        el.style.transform = "translateY(0)"; // snap back
        el.style.opacity = "1";
      }
    }

    el.addEventListener("pointerdown", onPointerDown);
    el.addEventListener("pointermove", onPointerMove);
    el.addEventListener("pointerup", onPointerUp);

    return () => {
      el.removeEventListener("pointerdown", onPointerDown);
      el.removeEventListener("pointermove", onPointerMove);
      el.removeEventListener("pointerup", onPointerUp);
    };
  }, [onDismiss]);

  return elementRef;
}
```

---

## 7. Animation Libraries

```javascript
// GSAP: industry-standard, very powerful timeline control
import gsap from "gsap";

gsap.to(".box", {
  x: 100,
  rotation: 360,
  duration: 1,
  ease: "power2.out",
});

// Timeline: sequence multiple animations
const tl = gsap.timeline();
tl.to(".box1", { x: 100, duration: 0.5 })
  .to(".box2", { y: 50, duration: 0.5 }, "-=0.25") // overlap by 0.25s
  .to(".box3", { opacity: 0, duration: 0.3 });

// Framer Motion: React-first, declarative
import { motion, AnimatePresence } from "framer-motion";

function Card({ isVisible }) {
  return (
    <AnimatePresence>
      {isVisible && (
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: -20 }}
          transition={{ duration: 0.3, ease: "easeOut" }}
        >
          Content
        </motion.div>
      )}
    </AnimatePresence>
  );
}

// Motion One: lightweight, built on WAAPI
import { animate, stagger } from "motion";

animate(
  ".item",
  { opacity: [0, 1], y: [20, 0] },
  { delay: stagger(0.05), duration: 0.3, easing: "ease-out" },
);

// react-spring: physics-based, hooks API
import { useSpring, animated } from "@react-spring/web";

function Box() {
  const [hovered, setHovered] = useState(false);
  const springs = useSpring({
    scale: hovered ? 1.1 : 1,
    config: { tension: 300, friction: 20 },
  });
  return (
    <animated.div
      style={springs}
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
    >
      Hover me
    </animated.div>
  );
}
```

---

## 8. Framer Motion Patterns

```jsx
// Layout animations: automatic FLIP-based transitions
function ExpandableCard({ isExpanded, children }) {
  return (
    <motion.div layout className="card" transition={{ duration: 0.3 }}>
      <motion.h2 layout="position">Title</motion.h2>
      {isExpanded && (
        <motion.div
          initial={{ opacity: 0, height: 0 }}
          animate={{ opacity: 1, height: "auto" }}
          exit={{ opacity: 0, height: 0 }}
        >
          {children}
        </motion.div>
      )}
    </motion.div>
  );
}

// Shared layout animations (magic move between components)
function Gallery({ selectedId, items }) {
  return (
    <LayoutGroup>
      <div className="grid">
        {items.map((item) => (
          <motion.img
            key={item.id}
            layoutId={`image-${item.id}`} // shared ID enables FLIP transition
            src={item.thumb}
            onClick={() => setSelectedId(item.id)}
          />
        ))}
      </div>
      <AnimatePresence>
        {selectedId && (
          <motion.div className="lightbox">
            <motion.img
              layoutId={`image-${selectedId}`} // same ID — animates FROM grid position
              src={items.find((i) => i.id === selectedId).full}
            />
          </motion.div>
        )}
      </AnimatePresence>
    </LayoutGroup>
  );
}

// Gesture animations
function DraggableCard() {
  return (
    <motion.div
      drag
      dragConstraints={{ left: 0, right: 0, top: 0, bottom: 0 }}
      dragElastic={0.2}
      whileDrag={{ scale: 1.05 }}
      whileTap={{ scale: 0.95 }}
    >
      Drag me
    </motion.div>
  );
}

// Variants: coordinated multi-element animations
const containerVariants = {
  hidden: { opacity: 0 },
  visible: { opacity: 1, transition: { staggerChildren: 0.1 } },
};
const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: { opacity: 1, y: 0 },
};

function StaggeredList({ items }) {
  return (
    <motion.ul variants={containerVariants} initial="hidden" animate="visible">
      {items.map((item) => (
        <motion.li key={item.id} variants={itemVariants}>
          {item.name}
        </motion.li>
      ))}
    </motion.ul>
  );
}
```

---

## 9. Scroll-Driven Animation

```css
/* Native CSS scroll-driven animations (Chrome 115+) */
@keyframes fadeInOnScroll {
  from {
    opacity: 0;
    transform: translateY(40px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.scroll-reveal {
  animation: fadeInOnScroll linear both;
  animation-timeline: view(); /* tied to element's visibility in viewport */
  animation-range: entry 0% cover 30%; /* starts at entry, completes at 30% covered */
}

/* Progress bar tied to page scroll */
.scroll-progress {
  transform-origin: left;
  animation: scaleProgress linear;
  animation-timeline: scroll(root); /* tied to document scroll position */
}
@keyframes scaleProgress {
  from {
    transform: scaleX(0);
  }
  to {
    transform: scaleX(1);
  }
}
```

```javascript
// JavaScript fallback: scroll-linked animation via IntersectionObserver + rAF
function useScrollProgress(elementRef) {
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    let rafId;
    function update() {
      const el = elementRef.current;
      if (!el) return;
      const rect = el.getBoundingClientRect();
      const vh = window.innerHeight;
      const raw = 1 - (rect.top + rect.height) / (vh + rect.height);
      setProgress(Math.max(0, Math.min(1, raw)));
      rafId = requestAnimationFrame(update);
    }
    rafId = requestAnimationFrame(update);
    return () => cancelAnimationFrame(rafId);
  }, [elementRef]);

  return progress;
}

// Use WAAPI's ScrollTimeline (progressive enhancement)
if ("ScrollTimeline" in window) {
  element.animate(
    { opacity: [0, 1], transform: ["translateY(40px)", "translateY(0)"] },
    {
      timeline: new ScrollTimeline({ source: document.documentElement }),
      rangeStart: "cover 0%",
      rangeEnd: "cover 50%",
    },
  );
}
```

---

## 10. Canvas and WebGL Animation

```javascript
// Canvas: full control, runs on main thread (or OffscreenCanvas in worker)
function animateCanvas(canvas) {
  const ctx = canvas.getContext("2d");
  let particles = createParticles(100);

  function render(timestamp) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    particles.forEach((p) => {
      p.x += p.vx;
      p.y += p.vy;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
      ctx.fillStyle = p.color;
      ctx.fill();
    });

    requestAnimationFrame(render);
  }
  requestAnimationFrame(render);
}

// OffscreenCanvas in a Worker: animation off the main thread entirely
// main.js
const canvas = document.querySelector("canvas");
const offscreen = canvas.transferControlToOffscreen();
const worker = new Worker("canvas-worker.js");
worker.postMessage({ canvas: offscreen }, [offscreen]);

// canvas-worker.js
self.onmessage = ({ data }) => {
  const ctx = data.canvas.getContext("2d");
  function render() {
    // Animation logic runs entirely in worker — zero main thread cost
    requestAnimationFrame(render);
  }
  render();
};
```

---

## 11. Good Practices

### ✅ Use Web Animations API for compositor-backed JS animation

```javascript
// ✅ WAAPI runs on compositor when animating transform/opacity — same perf as CSS
element.animate([{ transform: "scale(1)" }, { transform: "scale(1.1)" }], {
  duration: 200,
  easing: "ease-out",
});
```

### ✅ Always clean up rAF loops

```javascript
useEffect(() => {
  let id = requestAnimationFrame(loop);
  function loop() {
    /* ... */ id = requestAnimationFrame(loop);
  }
  return () => cancelAnimationFrame(id); // ✅ prevent leaks
}, []);
```

### ✅ Use spring physics for interruptible/gesture animations

```javascript
// ✅ Springs naturally handle interruption (changing target mid-flight)
// Time-based easing curves look jarring if interrupted mid-animation
```

---

## 12. Bad Practices

### ❌ Animating via setInterval

```javascript
// ❌ setInterval doesn't sync with display refresh, drifts over time
setInterval(() => {
  position += 1;
  element.style.left = position + "px";
}, 16); // approximately 60fps but NOT synced to actual repaints

// ✅ requestAnimationFrame syncs to display refresh
function loop() {
  position += 1;
  element.style.left = position + "px";
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
```

### ❌ Layout reads inside the animation loop

```javascript
// ❌ Forces layout every frame
function loop() {
  const height = element.offsetHeight; // forces layout if dirty
  element.style.transform = `translateY(${height}px)`;
  requestAnimationFrame(loop);
}

// ✅ Read once, outside the loop
const height = element.offsetHeight;
function loop() {
  element.style.transform = `translateY(${height}px)`;
  requestAnimationFrame(loop);
}
```

---

## 13. Common Mistakes

### Mistake 1 — Not accounting for variable frame timing

```javascript
// ❌ Assumes constant 16.7ms per frame — breaks on variable refresh rate displays
function loop() {
  position += 2; // 2px per frame: speed varies with display refresh rate!
  requestAnimationFrame(loop);
}

// ✅ Use deltaTime for frame-rate-independent speed
let last = null;
function loop(now) {
  if (last === null) last = now;
  const dt = now - last;
  last = now;
  position += 0.12 * dt; // px per ms: consistent speed regardless of refresh rate
  requestAnimationFrame(loop);
}
```

### Mistake 2 — Memory leak from uncancelled WAAPI animations

```javascript
// ❌ Animation references element even after component unmounts
useEffect(() => {
  element.animate(keyframes, { duration: Infinity, iterations: Infinity });
  // No cleanup — animation keeps running, element stays referenced
}, []);

// ✅ Store and cancel on cleanup
useEffect(() => {
  const anim = element.animate(keyframes, {
    duration: Infinity,
    iterations: Infinity,
  });
  return () => anim.cancel();
}, []);
```

---

## 14. Interview-Level Explanation

> **"When would you use JavaScript animation instead of CSS? What's the Web Animations API?"**

**Strong answer:**

> "CSS handles the majority of UI animation well, but JavaScript becomes necessary when the animation needs runtime logic that CSS can't express: physics-based motion that responds to velocity, animations driven by gesture input like drag-to-dismiss, animations that need to be paused, reversed, or scrubbed programmatically, or animations synchronized with arbitrary application state.
>
> The Web Animations API is the modern way to do JavaScript-driven animation without sacrificing performance. `element.animate()` takes keyframes and options similar to CSS `@keyframes`, but returns an Animation object you can control: pause, play, reverse, set playback rate, or scrub to a specific time via `currentTime`. Critically, when animating `transform` and `opacity`, WAAPI runs on the compositor thread just like CSS animations — you get the JavaScript control without the main-thread performance cost of manually setting styles every frame via requestAnimationFrame.
>
> For cases that genuinely need per-frame logic — like spring physics — requestAnimationFrame is necessary. A spring simulation calculates a damped harmonic oscillator: spring force pulls toward the target, a damping force opposes velocity, and the current position updates each frame based on accumulated velocity. The key advantage of springs over time-based easing curves is that they handle interruption gracefully — if you change the target mid-animation, like a user dragging in the opposite direction, the spring naturally redirects without a visible discontinuity. A duration-based ease curve, if interrupted and given a new destination, requires manually computing a new curve from the current position, which often produces visible jank.
>
> For gesture-driven UI, I track pointer events directly — pointerdown captures the start position, pointermove updates the element's transform in real-time, and pointerup decides whether to commit or reverse based on either the distance dragged or the velocity at release, calculated from the time delta between recent pointer move events.
>
> Production apps typically reach for a library — Framer Motion for React with its declarative `initial`/`animate`/`exit` API and powerful layout animations using the FLIP technique, or GSAP for non-React projects needing fine-grained timeline sequencing. These libraries handle the edge cases — animation interruption, cleanup, batching — that are easy to get wrong building from scratch."

---

## 15. Exercises

### Exercise 1 — Build a number counter animation

Implement a reusable function that animates a number from a starting value to a target value over a given duration, using ease-out easing, callable from React.

<details>
<summary>Solution</summary>

```typescript
function easeOutCubic(t: number): number {
  return 1 - Math.pow(1 - t, 3);
}

function useAnimatedCounter(targetValue: number, duration = 1000) {
  const [displayValue, setDisplayValue] = useState(targetValue);
  const previousValue = useRef(targetValue);
  const rafRef = useRef<number>();

  useEffect(() => {
    const from = previousValue.current;
    const to   = targetValue;

    if (from === to) return;

    const startTime = performance.now();

    function tick(now: number) {
      const elapsed  = now - startTime;
      const progress = Math.min(elapsed / duration, 1);
      const eased    = easeOutCubic(progress);
      const current  = from + (to - from) * eased;

      setDisplayValue(Math.round(current));

      if (progress < 1) {
        rafRef.current = requestAnimationFrame(tick);
      } else {
        previousValue.current = to;
      }
    }

    rafRef.current = requestAnimationFrame(tick);

    return () => {
      if (rafRef.current) cancelAnimationFrame(rafRef.current);
    };
  }, [targetValue, duration]);

  return displayValue;
}

// Usage:
function StatCounter({ value }: { value: number }) {
  const animated = useAnimatedCounter(value, 800);
  return <span>{animated.toLocaleString()}</span>;
}
```

</details>

---

## 🔗 Related Topics

- [`animations/01-css-animations.md`](./01-css-animations.md) — CSS-native animation primitives
- [`animations/03-compositor-animations.md`](./03-compositor-animations.md) — How animations reach the compositor
- [`rendering/03-cooperative-scheduling.md`](../rendering/03-cooperative-scheduling.md) — rAF in the scheduling pipeline
- [`performance/04-raf-optimization.md`](../performance/04-raf-optimization.md) — rAF performance patterns

---

<div align="center">

**Next:** [`animations/03-compositor-animations.md`](./03-compositor-animations.md) →

</div>
