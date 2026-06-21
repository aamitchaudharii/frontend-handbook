# 01 — CSS Animations and Transitions

> **"CSS animations are a contract with the browser: you describe the start state, the end state, and the timing, and the browser figures out every frame in between. The moment you fight that contract — animating properties that trigger layout, fighting the timing function, synchronizing dozens of competing transitions — is the moment your animations stop feeling smooth."**

CSS animations and transitions are the browser's native animation primitives, running independently of JavaScript and, for the right properties, independently of the main thread entirely. This document covers the mechanics of transitions and keyframe animations, easing functions, the animation-fill-mode and timing model, performance implications, and the patterns that produce animations indistinguishable from native app polish.

---

## 📚 Table of Contents

1. [Transitions vs Animations](#1-transitions-vs-animations)
2. [The Transition Property Deep Dive](#2-the-transition-property-deep-dive)
3. [Keyframe Animations](#3-keyframe-animations)
4. [Timing Functions and Easing](#4-timing-functions-and-easing)
5. [Animation Fill Mode and Direction](#5-animation-fill-mode-and-direction)
6. [Animating Custom Properties](#6-animating-custom-properties)
7. [Staggered Animations](#7-staggered-animations)
8. [Animation Events](#8-animation-events)
9. [Reduced Motion](#9-reduced-motion)
10. [Performance Implications](#10-performance-implications)
11. [Common Animation Patterns](#11-common-animation-patterns)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. Transitions vs Animations

```
CSS TRANSITIONS:
  Trigger:    State change (hover, class toggle, attribute change)
  Direction:  Always from current value to new value
  Control:    Limited (can't loop, can't have multiple keyframes)
  Use for:    Simple state changes (hover effects, toggles)

CSS ANIMATIONS (keyframes):
  Trigger:    Automatic (on apply) or class-triggered
  Direction:  Defined by keyframes — full control over the sequence
  Control:    Full (loop, alternate, multiple steps, delays)
  Use for:    Complex sequences, looping animations, entrances/exits

DECISION:
  2 states (A → B): use transition
  Multiple states or loop: use @keyframes animation
```

---

## 2. The Transition Property Deep Dive

```css
/* Full transition shorthand */
.button {
  transition:
    background-color 0.3s ease-in-out,
    transform 0.2s cubic-bezier(0.4, 0, 0.2, 1),
    box-shadow 0.3s ease 0.1s; /* property duration timing-function delay */
}

/* Individual longhand properties */
.button {
  transition-property: background-color, transform, box-shadow;
  transition-duration: 0.3s, 0.2s, 0.3s;
  transition-timing-function: ease-in-out, cubic-bezier(0.4, 0, 0.2, 1), ease;
  transition-delay: 0s, 0s, 0.1s;
}

/* Transition all properties (use cautiously — performance and unintended transitions) */
.button {
  transition: all 0.3s ease;
  /* ❌ Risk: transitions properties you didn't intend (layout shifts, color changes
     from unrelated CSS rule changes) */
}

/* Better: explicit property list */
.button {
  transition:
    transform 0.3s ease,
    opacity 0.3s ease;
  /* ✅ Only animates what you intend */
}
```

### Transition Behavior Details

```css
/* Transitions only fire on property VALUE changes, not initial render */
.card {
  opacity: 0;
  transition: opacity 0.3s;
}
/* Adding .card to DOM: opacity starts at 0, NO transition (no prior value to transition from) */

.card.visible {
  opacity: 1;
}
/* Adding .visible class AFTER initial render: transition fires 0 → 1 */

/* To trigger transition on mount: ensure two renders happen */
function FadeIn({ children }) {
  const [visible, setVisible] = useState(false);
  useEffect(() => {
    // Force a frame between mount (opacity:0) and class toggle (opacity:1)
    requestAnimationFrame(() => requestAnimationFrame(() => setVisible(true)));
  }, []);
  return <div className={visible ? 'card visible' : 'card'}>{children}</div>;
}
```

### transition-behavior: allow-discrete

```css
/* Modern: transition discrete properties like display (Chrome 117+) */
.modal {
  display: none;
  opacity: 0;
  transition:
    opacity 0.3s,
    display 0.3s allow-discrete;
}
.modal.open {
  display: block;
  opacity: 1;
}
/* allow-discrete: display switches at the START of the transition when
   showing, and at the END when hiding — enables fade + display:none combo
   without needing JS to delay the display change */

@starting-style {
  .modal.open {
    opacity: 0; /* defines the "from" state for elements that just appeared */
  }
}
```

---

## 3. Keyframe Animations

```css
/* Define keyframes */
@keyframes slideInFade {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Multi-step keyframes */
@keyframes pulse {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.05);
    opacity: 0.8;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

/* Apply animation */
.notification {
  animation: slideInFade 0.4s ease-out;
}

.loading-dot {
  animation: pulse 1.5s ease-in-out infinite;
}

/* Full animation shorthand */
.element {
  animation: name slideInFade duration 0.4s timing-function ease-out delay 0s
    iteration-count 1 direction normal fill-mode forwards play-state running;
  /* Order matters for duration vs delay (first time value = duration) */
  animation: slideInFade 0.4s ease-out 0s 1 normal forwards running;
}
```

### Multiple Simultaneous Animations

```css
/* Apply multiple animations to one element */
.complex-element {
  animation:
    fadeIn 0.3s ease-out,
    slideUp 0.4s ease-out,
    rotate 2s linear infinite;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
@keyframes slideUp {
  from {
    transform: translateY(20px);
  }
  to {
    transform: translateY(0);
  }
}
@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* WARNING: multiple animations targeting the SAME property conflict
   Last one wins for that property (transform here would only show rotate,
   not slideUp, since both animate transform) */

/* Fix: combine into one keyframe set if animating the same property */
@keyframes slideUpAndRotate {
  from {
    transform: translateY(20px) rotate(0deg);
  }
  to {
    transform: translateY(0) rotate(360deg);
  }
}
```

---

## 4. Timing Functions and Easing

```css
/* Built-in easing functions */
.linear {
  transition-timing-function: linear;
} /* constant speed */
.ease {
  transition-timing-function: ease;
} /* default: slow-fast-slow */
.ease-in {
  transition-timing-function: ease-in;
} /* slow start */
.ease-out {
  transition-timing-function: ease-out;
} /* slow end (most natural for UI) */
.ease-in-out {
  transition-timing-function: ease-in-out;
} /* slow start AND end */

/* Custom cubic-bezier curves */
.custom {
  transition-timing-function: cubic-bezier(
    0.4,
    0,
    0.2,
    1
  ); /* Material Design "standard" easing */
}

/* Common named curves (as cubic-bezier): */
/* Material Design standard: cubic-bezier(0.4, 0.0, 0.2, 1)   */
/* Material Design decelerate: cubic-bezier(0.0, 0.0, 0.2, 1) */
/* Material Design accelerate: cubic-bezier(0.4, 0.0, 1, 1)   */
/* iOS default: cubic-bezier(0.25, 0.1, 0.25, 1)               */
/* Bounce-ish overshoot: cubic-bezier(0.34, 1.56, 0.64, 1)     */

/* Step functions: discrete jumps instead of smooth interpolation */
.steps {
  animation: spriteAnimation 1s steps(8) infinite;
  /* 8 discrete frames — useful for sprite sheet animations */
}

@keyframes spriteAnimation {
  from {
    background-position: 0 0;
  }
  to {
    background-position: -800px 0;
  } /* 8 frames × 100px each */
}

/* linear() function: custom easing with multiple points (Chrome 113+) */
.spring-like {
  transition-timing-function: linear(
    0,
    0.063,
    0.25,
    0.563,
    1,
    0.812,
    0.625,
    0.812,
    1,
    0.937,
    1
  );
  /* Approximates a spring physics curve using interpolated points */
}
```

### Choosing the Right Easing

```
ease-out: BEST DEFAULT for UI elements entering or responding to user action
  → Starts fast (immediate response), ends slow (settles naturally)
  → Matches user expectation: action causes immediate visible response

ease-in: for elements LEAVING the screen
  → Starts slow, accelerates away
  → Element seems to "speed off"

ease-in-out: for elements moving WITHOUT user-initiated start/stop
  → Smooth, symmetric — good for looping/ambient animations

linear: AVOID for most UI (feels robotic)
  → Use for: progress bars, rotating spinners, marquees (literal constant motion)

cubic-bezier with overshoot (bounce): for playful/attention-grabbing UI
  → Use sparingly — overuse feels gimmicky
```

---

## 5. Animation Fill Mode and Direction

```css
/* animation-fill-mode: what happens before/after the animation */
.element {
  animation: slideIn 0.5s ease-out;
  animation-fill-mode: none; /* default: element returns to original CSS state after animation */
}

.element-forwards {
  animation: slideIn 0.5s ease-out forwards;
  /* forwards: element KEEPS the final keyframe's styles after animation ends */
  /* Use for: entrance animations where the end state should persist */
}

.element-backwards {
  animation: slideIn 0.5s ease-out 1s backwards;
  /* backwards: element applies the FIRST keyframe's styles during the delay period */
  /* Use for: avoiding flash of unstyled content during animation-delay */
}

.element-both {
  animation: slideIn 0.5s ease-out 1s both;
  /* both: combines forwards and backwards behavior */
}

/* animation-direction: playback direction */
.normal {
  animation-direction: normal;
} /* 0% → 100% */
.reverse {
  animation-direction: reverse;
} /* 100% → 0% */
.alternate {
  animation-direction: alternate;
} /* 0%→100%, then 100%→0%, repeating */
.alt-reverse {
  animation-direction: alternate-reverse;
} /* 100%→0%, then 0%→100% */

/* Practical pulse animation using alternate (no jump back) */
.pulse {
  animation: pulseScale 1s ease-in-out infinite alternate;
}
@keyframes pulseScale {
  from {
    transform: scale(1);
  }
  to {
    transform: scale(1.1);
  }
}
/* alternate: smoothly scales up THEN down, no jarring reset to scale(1) */
```

---

## 6. Animating Custom Properties

```css
/* Custom properties (CSS variables) CAN be animated, but require @property
   registration for smooth interpolation of non-string values */

@property --progress {
  syntax: "<number>";
  inherits: false;
  initial-value: 0;
}

.progress-ring {
  --progress: 0;
  background: conic-gradient(#4fc3f7 calc(var(--progress) * 1%), #e0e0e0 0);
  transition: --progress 0.5s ease-out;
}

.progress-ring.loaded {
  --progress: 75; /* animates smoothly from 0 to 75 due to @property registration */
}

/* Without @property: custom properties don't interpolate (jump instantly) */
.no-property-registration {
  --opacity: 0;
  opacity: var(--opacity);
  transition: --opacity 0.5s; /* doesn't work without @property — treated as discrete string */
}

/* @property also enables animating gradients, angles smoothly */
@property --angle {
  syntax: "<angle>";
  inherits: false;
  initial-value: 0deg;
}

.spinning-gradient {
  --angle: 0deg;
  background: conic-gradient(from var(--angle), red, blue, red);
  animation: spin 3s linear infinite;
}
@keyframes spin {
  to {
    --angle: 360deg;
  }
}
```

---

## 7. Staggered Animations

```css
/* Stagger using nth-child with animation-delay */
.list-item {
  opacity: 0;
  animation: slideInFade 0.4s ease-out forwards;
}
.list-item:nth-child(1) {
  animation-delay: 0ms;
}
.list-item:nth-child(2) {
  animation-delay: 50ms;
}
.list-item:nth-child(3) {
  animation-delay: 100ms;
}
.list-item:nth-child(4) {
  animation-delay: 150ms;
}
/* ... tedious to maintain for long lists */
```

```javascript
// Dynamic stagger via inline custom property
function StaggeredList({ items }) {
  return (
    <ul>
      {items.map((item, i) => (
        <li
          key={item.id}
          className="list-item"
          style={{ '--stagger-index': i } as React.CSSProperties}
        >
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

```css
.list-item {
  opacity: 0;
  animation: slideInFade 0.4s ease-out forwards;
  animation-delay: calc(var(--stagger-index) * 50ms);
}
```

```css
/* CSS-only stagger using calc() and nth-child for shorter lists */
.grid-item {
  opacity: 0;
  animation: fadeIn 0.3s ease-out forwards;
}
@for $i from 1 through 20 {
  .grid-item:nth-child(#{$i}) {
    animation-delay: #{$i * 30}ms; /* SCSS loop generates delays */
  }
}
```

---

## 8. Animation Events

```javascript
// Listen for animation lifecycle events
const element = document.querySelector(".animated");

element.addEventListener("animationstart", (e) => {
  console.log(`Animation started: ${e.animationName}`);
});

element.addEventListener("animationend", (e) => {
  console.log(
    `Animation ended: ${e.animationName}, elapsed: ${e.elapsedTime}s`,
  );
  // Common use: remove the animation class after it completes
  element.classList.remove("shake");
});

element.addEventListener("animationiteration", (e) => {
  console.log(`Iteration completed: ${e.animationName}`);
  // Fires on each loop of an infinite/repeating animation
});

element.addEventListener("animationcancel", (e) => {
  // Fires when animation is cancelled (element removed, animation-name changed)
  console.log("Animation cancelled");
});

// Transition events (same pattern)
element.addEventListener("transitionend", (e) => {
  console.log(`Transition ended: ${e.propertyName}`);
});
```

### React: Animation Completion Handling

```jsx
// Remove element from DOM after exit animation completes
function AnimatedListItem({ item, onRemove }) {
  const [isExiting, setIsExiting] = useState(false);

  const handleRemove = () => setIsExiting(true);

  const handleAnimationEnd = (e) => {
    if (e.animationName === "slideOutFade") {
      onRemove(item.id); // actually remove from state/DOM
    }
  };

  return (
    <li
      className={isExiting ? "exiting" : ""}
      onAnimationEnd={handleAnimationEnd}
    >
      {item.name}
      <button onClick={handleRemove}>×</button>
    </li>
  );
}
```

```css
.exiting {
  animation: slideOutFade 0.3s ease-in forwards;
}
@keyframes slideOutFade {
  to {
    opacity: 0;
    transform: translateX(-20px);
  }
}
```

---

## 9. Reduced Motion

```css
/* Respect user's motion preference */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* Better: provide a reduced (not eliminated) experience */
.modal {
  animation: slideIn 0.3s ease-out;
}

@media (prefers-reduced-motion: reduce) {
  .modal {
    animation: fadeIn 0.15s ease-out; /* simpler fade instead of slide+fade */
  }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
```

```javascript
// JavaScript: respect reduced motion in animation logic
const prefersReducedMotion = window.matchMedia(
  "(prefers-reduced-motion: reduce)",
).matches;

function animateElement(element) {
  if (prefersReducedMotion) {
    element.style.opacity = "1"; // instant, no animation
    return;
  }
  element.animate(
    [
      { opacity: 0, transform: "translateY(20px)" },
      { opacity: 1, transform: "translateY(0)" },
    ],
    { duration: 400, easing: "ease-out", fill: "forwards" },
  );
}

// Listen for preference changes (user can toggle OS setting while page is open)
window
  .matchMedia("(prefers-reduced-motion: reduce)")
  .addEventListener("change", (e) => {
    document.documentElement.classList.toggle("reduce-motion", e.matches);
  });
```

---

## 10. Performance Implications

```css
/* ✅ GPU-accelerated: transform and opacity only */
.efficient {
  transition:
    transform 0.3s,
    opacity 0.3s;
}
.efficient:hover {
  transform: translateY(-4px) scale(1.02);
  opacity: 0.9;
}

/* ❌ Triggers layout: animating box-model properties */
.inefficient {
  transition:
    width 0.3s,
    height 0.3s,
    margin 0.3s;
}
.inefficient:hover {
  width: 120%; /* layout + paint + composite every frame */
  height: 120%;
  margin: 20px;
}

/* ✅ Promote to its own layer for complex/frequent animations */
.frequently-animated {
  will-change: transform, opacity;
}
/* Remove will-change after animation completes (apply dynamically via JS)
   to avoid permanently consuming GPU memory */
```

See [`rendering/04-paint-optimization.md`](../rendering/04-paint-optimization.md) for the full breakdown of which CSS properties are compositor-only vs paint-triggering vs layout-triggering.

---

## 11. Common Animation Patterns

### Skeleton Loading Shimmer

```css
.skeleton {
  background: linear-gradient(90deg, #e0e0e0 25%, #f0f0f0 50%, #e0e0e0 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s ease-in-out infinite;
}
@keyframes shimmer {
  0% {
    background-position: 200% 0;
  }
  100% {
    background-position: -200% 0;
  }
}
```

### Button Press Feedback

```css
.button {
  transition:
    transform 0.1s ease-out,
    box-shadow 0.1s ease-out;
}
.button:active {
  transform: scale(0.96);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}
```

### Attention-Grabbing Shake

```css
@keyframes shake {
  10%,
  90% {
    transform: translateX(-1px);
  }
  20%,
  80% {
    transform: translateX(2px);
  }
  30%,
  50%,
  70% {
    transform: translateX(-4px);
  }
  40%,
  60% {
    transform: translateX(4px);
  }
}
.input-error {
  animation: shake 0.5s ease-in-out;
  border-color: #e53935;
}
```

### Modal Entrance/Exit

```css
.modal-backdrop {
  opacity: 0;
  transition: opacity 0.2s ease-out;
}
.modal-backdrop.open {
  opacity: 1;
}

.modal-content {
  opacity: 0;
  transform: scale(0.95) translateY(10px);
  transition:
    opacity 0.2s ease-out,
    transform 0.2s ease-out;
}
.modal-content.open {
  opacity: 1;
  transform: scale(1) translateY(0);
}
```

---

## 12. Good Practices

### ✅ Animate transform and opacity for performance-critical UI

```css
/* ✅ Compositor-only properties */
.card:hover {
  transform: translateY(-2px);
  opacity: 0.95;
}
```

### ✅ Use ease-out for entrances, ease-in for exits

```css
.enter {
  animation: slideIn 0.3s ease-out;
}
.exit {
  animation: slideOut 0.2s ease-in;
}
```

### ✅ Always respect prefers-reduced-motion

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

### ✅ Keep durations short and purposeful

```
Micro-interactions (hover, press): 100-200ms
Element transitions (modal, dropdown): 200-300ms
Page transitions: 300-500ms
Avoid: anything over 500ms for UI feedback (feels sluggish)
```

---

## 13. Bad Practices

### ❌ `transition: all` everywhere

```css
/* ❌ Transitions properties you didn't intend, hurts performance */
* {
  transition: all 0.3s;
}
```

### ❌ Animating layout properties

```css
/* ❌ width/height/margin animations trigger layout every frame */
.bad {
  transition:
    width 0.3s,
    height 0.3s;
}
.bad:hover {
  width: 200px;
  height: 200px;
}

/* ✅ Use transform: scale instead */
.good {
  transition: transform 0.3s;
  transform-origin: top left;
}
.good:hover {
  transform: scale(1.5);
}
```

### ❌ Overly long or bouncy animations on every interaction

```css
/* ❌ 1 second bounce on every button hover = annoying */
.button:hover {
  animation: excessiveBounce 1s cubic-bezier(0.68, -0.55, 0.27, 1.55) infinite;
}
```

---

## 14. Common Mistakes

### Mistake 1 — Transition not firing on mount

```javascript
// ❌ Class added in same render — no transition (no prior state to transition from)
function FadeIn() {
  return <div className="fade-in visible">Content</div>; // already visible from start
}

// ✅ Two-step: render hidden, then toggle visible in next frame
function FadeIn() {
  const [visible, setVisible] = useState(false);
  useEffect(() => {
    requestAnimationFrame(() => setVisible(true));
  }, []);
  return <div className={`fade-in ${visible ? "visible" : ""}`}>Content</div>;
}
```

### Mistake 2 — Forgetting animation-fill-mode causes flicker

```css
/* ❌ Element snaps back to original state after animation completes */
.toast {
  animation: slideIn 0.3s ease-out;
  /* no fill-mode: after 0.3s, returns to pre-animation CSS (likely invisible) */
}

/* ✅ forwards keeps the final state */
.toast {
  animation: slideIn 0.3s ease-out forwards;
}
```

### Mistake 3 — Not cleaning up infinite animations

```javascript
// ❌ Infinite CSS animation continues even when element is off-screen
.spinner { animation: spin 1s linear infinite; }
// If there are 50 off-screen spinners: still consuming GPU cycles

// ✅ Pause animations for off-screen elements
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    entry.target.style.animationPlayState = entry.isIntersecting ? 'running' : 'paused';
  });
});
document.querySelectorAll('.spinner').forEach(el => observer.observe(el));
```

---

## 15. Interview-Level Explanation

> **"What's the difference between CSS transitions and animations? How do you ensure animations perform well?"**

**Strong answer:**

> "Transitions interpolate between two states triggered by a property change — hover, class toggle, attribute change. They're simple but limited to a single from-to interpolation with one timing function. Keyframe animations give full control over a sequence of states, support looping via `animation-iteration-count`, and can run automatically on element mount without needing a triggering state change.
>
> The performance principle is the same for both: stick to `transform` and `opacity`. These are compositor-only properties — the browser rasterizes the element once, uploads it as a GPU texture, and then each animation frame is just a matrix transform or alpha blend applied by the compositor thread. This runs independently of the main thread, so even if JavaScript is busy, these animations stay smooth at 60fps. Animating `width`, `height`, `margin`, or `top`/`left` triggers layout recalculation on every frame, which is orders of magnitude more expensive and can drop frames under load.
>
> For entrance and exit timing, `ease-out` feels natural for elements appearing — it starts fast, giving immediate visual feedback to the triggering action, then settles smoothly. `ease-in` suits exits — the element accelerates away. `linear` should be reserved for genuinely constant-rate motion like spinners or marquees, since it feels mechanical for most UI transitions.
>
> `animation-fill-mode: forwards` is essential for entrance animations — without it, the element snaps back to its pre-animation CSS state the instant the animation completes, causing a visible flicker.
>
> Accessibility requires respecting `prefers-reduced-motion`. Some users have vestibular disorders triggered by motion, and the media query lets you either disable animations entirely or substitute a simpler, non-disorienting transition like a plain fade instead of a slide-and-fade combination.
>
> For staggered list animations, I use a CSS custom property for the index combined with `calc()` for the delay, set inline via JavaScript when rendering the list — this scales to any list length without generating per-index CSS rules."

---

## 16. Exercises

### Exercise 1 — Build a notification toast system

Create CSS for a toast notification that: slides in from the right with fade, stays visible, then slides out to the right with fade on dismiss. Must respect reduced motion and use only compositor-friendly properties.

<details>
<summary>Solution</summary>

```css
.toast {
  position: fixed;
  top: 20px;
  right: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  padding: 16px 20px;
  opacity: 0;
  transform: translateX(120%);
  animation: toastIn 0.3s cubic-bezier(0.4, 0, 0.2, 1) forwards;
}

.toast.exiting {
  animation: toastOut 0.25s cubic-bezier(0.4, 0, 1, 1) forwards;
}

@keyframes toastIn {
  from {
    opacity: 0;
    transform: translateX(120%);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes toastOut {
  from {
    opacity: 1;
    transform: translateX(0);
  }
  to {
    opacity: 0;
    transform: translateX(120%);
  }
}

@media (prefers-reduced-motion: reduce) {
  .toast {
    animation: toastInReduced 0.15s ease-out forwards;
  }
  .toast.exiting {
    animation: toastOutReduced 0.15s ease-out forwards;
  }
  @keyframes toastInReduced {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }
  @keyframes toastOutReduced {
    from {
      opacity: 1;
    }
    to {
      opacity: 0;
    }
  }
}
```

```jsx
function Toast({ message, onDismiss }) {
  const [exiting, setExiting] = useState(false);

  const handleDismiss = () => setExiting(true);

  const handleAnimationEnd = (e) => {
    if (
      e.animationName === "toastOut" ||
      e.animationName === "toastOutReduced"
    ) {
      onDismiss();
    }
  };

  return (
    <div
      className={`toast ${exiting ? "exiting" : ""}`}
      onAnimationEnd={handleAnimationEnd}
    >
      {message}
      <button onClick={handleDismiss}>×</button>
    </div>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`animations/02-javascript-animations.md`](./02-javascript-animations.md) — Web Animations API and JS-driven animation
- [`rendering/04-paint-optimization.md`](../rendering/04-paint-optimization.md) — Why transform/opacity are fast
- [`animations/03-compositor-animations.md`](./03-compositor-animations.md) — Deep dive into the compositor thread
- [`browser-internals/06-composite-layers.md`](../browser-internals/06-composite-layers.md) — Layer promotion mechanics

---

<div align="center">

**Next:** [`animations/02-javascript-animations.md`](./02-javascript-animations.md) →

</div>
