# 04 — Micro-Interactions

> **"A micro-interaction is the difference between a button that 'works' and a button that feels alive. Users can't articulate why one interface feels polished and another feels cheap — but the answer is almost always in the hundred-millisecond details: the way a checkbox settles into place, the way a toast slides in just enough to notice, the way a button compresses slightly under a click."**

Micro-interactions are small, purposeful moments of feedback triggered by a single user action: a click, a toggle, a successful save, a validation error, a hover. They're typically under 400ms, often under 200ms, and their entire purpose is to communicate state change, provide feedback, and guide attention — without becoming the focus themselves. This document covers the anatomy of a micro-interaction, timing and easing conventions, the most common patterns, accessibility considerations, and the engineering discipline for keeping them performant at scale.

---

## 📚 Table of Contents

1. [The Anatomy of a Micro-Interaction](#1-the-anatomy-of-a-micro-interaction)
2. [Timing Conventions](#2-timing-conventions)
3. [Button and Click Feedback](#3-button-and-click-feedback)
4. [Form Field Interactions](#4-form-field-interactions)
5. [Toggle and Checkbox Patterns](#5-toggle-and-checkbox-patterns)
6. [Loading and Progress Feedback](#6-loading-and-progress-feedback)
7. [Success and Error Feedback](#7-success-and-error-feedback)
8. [Hover and Focus States](#8-hover-and-focus-states)
9. [Notification and Toast Patterns](#9-notification-and-toast-patterns)
10. [Haptic-Like Visual Feedback](#10-haptic-like-visual-feedback)
11. [Accessibility in Micro-Interactions](#11-accessibility-in-micro-interactions)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Anatomy of a Micro-Interaction

```
FOUR PARTS OF EVERY MICRO-INTERACTION:

1. TRIGGER
   What initiates it: user action (click, hover, type) or system event (data loaded, error occurred)

2. RULES
   What happens: what changes, in what order, how long it takes

3. FEEDBACK
   What the user perceives: visual, auditory, or haptic confirmation that something happened

4. LOOPS & MODES
   What happens on repeat or in edge cases: does it repeat? does state persist? what about rapid re-triggers?

EXAMPLE — "Like" button:
  Trigger:  User taps the heart icon
  Rules:    Icon fills, scales up briefly, then settles; count increments
  Feedback: Visual (fill + scale), optional haptic (mobile), color change
  Loops:    Tapping again un-likes (reverse animation); rapid taps are debounced
```

---

## 2. Timing Conventions

```
DURATION GUIDELINES (from Material Design, Apple HIG, and field-tested UX research):

INSTANT FEEDBACK (0-100ms):
  Button press/active state, ripple start
  Should feel immediate — any delay here feels laggy

QUICK TRANSITIONS (100-200ms):
  Hover states, focus rings, small icon animations,
  toggle switches, checkbox checks

STANDARD TRANSITIONS (200-300ms):
  Dropdown open/close, tooltip appearance, accordion expand,
  modal entrance, card hover lift

DELIBERATE TRANSITIONS (300-500ms):
  Page transitions, large modal/drawer entrances,
  multi-step reveals, complex layout shifts

AVOID (>500ms) for interactive feedback:
  Feels sluggish and makes the UI seem unresponsive
  Exception: deliberate "ta-da" moments (onboarding completion, achievement unlocked)
  where the delay itself is part of the desired emotional effect

EASING CONVENTIONS:
  Entering/appearing: ease-out (fast start, slow settle) — feels responsive
  Exiting/disappearing: ease-in (slow start, fast exit) — feels like it's "leaving"
  Moving/repositioning: ease-in-out (smooth both ends) — feels natural
  Bouncy/playful: cubic-bezier with overshoot — use sparingly, high-engagement moments only
```

---

## 3. Button and Click Feedback

```css
/* Standard button: press feedback + hover lift */
.btn {
  transform: translateY(0) scale(1);
  transition:
    transform 0.1s ease-out,
    box-shadow 0.15s ease-out,
    background-color 0.15s ease-out;
}

.btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.btn:active {
  transform: translateY(0) scale(0.97); /* compress on press */
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
  transition-duration: 0.05s; /* faster on press — feels more responsive */
}
```

### Material Ripple Effect

```jsx
function RippleButton({ children, onClick }) {
  const [ripples, setRipples] = useState([]);

  const handleClick = (e) => {
    const rect = e.currentTarget.getBoundingClientRect();
    const size = Math.max(rect.width, rect.height) * 2;
    const x = e.clientX - rect.left - size / 2;
    const y = e.clientY - rect.top - size / 2;
    const id = Date.now();

    setRipples((prev) => [...prev, { id, x, y, size }]);
    setTimeout(() => {
      setRipples((prev) => prev.filter((r) => r.id !== id));
    }, 600);

    onClick?.(e);
  };

  return (
    <button className="ripple-btn" onClick={handleClick}>
      {children}
      {ripples.map((r) => (
        <span
          key={r.id}
          className="ripple"
          style={{ left: r.x, top: r.y, width: r.size, height: r.size }}
        />
      ))}
    </button>
  );
}
```

```css
.ripple-btn {
  position: relative;
  overflow: hidden;
}
.ripple {
  position: absolute;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.5);
  transform: scale(0);
  animation: rippleEffect 0.6s ease-out;
  pointer-events: none;
}
@keyframes rippleEffect {
  to {
    transform: scale(1);
    opacity: 0;
  }
}
```

---

## 4. Form Field Interactions

```css
/* Floating label pattern */
.field {
  position: relative;
}
.field label {
  position: absolute;
  left: 12px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 16px;
  color: #888;
  pointer-events: none;
  transition:
    top 0.2s ease-out,
    font-size 0.2s ease-out,
    color 0.2s ease-out;
}
.field input {
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 16px 12px 8px;
  transition: border-color 0.2s ease-out;
}
.field input:focus {
  border-color: #4fc3f7;
  outline: none;
}
.field input:focus + label,
.field input:not(:placeholder-shown) + label {
  top: 8px;
  font-size: 12px;
  color: #4fc3f7;
}
```

```css
/* Validation feedback */
.field.has-error input {
  border-color: #e53935;
  animation: fieldShake 0.4s ease-in-out;
}
@keyframes fieldShake {
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
    transform: translateX(-3px);
  }
  40%,
  60% {
    transform: translateX(3px);
  }
}
.field-error-message {
  color: #e53935;
  font-size: 13px;
  max-height: 0;
  opacity: 0;
  overflow: hidden;
  transition:
    max-height 0.2s ease-out,
    opacity 0.2s ease-out;
}
.field.has-error .field-error-message {
  max-height: 30px;
  opacity: 1;
}
```

### Character Counter with Approaching-Limit Feedback

```jsx
function CharacterCounter({ value, max }) {
  const remaining = max - value.length;
  const isNearLimit = remaining <= 20 && remaining > 0;
  const isOverLimit = remaining < 0;

  return (
    <span
      className={`char-counter ${isNearLimit ? "near-limit" : ""} ${isOverLimit ? "over-limit" : ""}`}
      aria-live="polite"
    >
      {remaining} characters remaining
    </span>
  );
}
```

```css
.char-counter {
  transition: color 0.2s ease-out;
}
.char-counter.near-limit {
  color: #f9a825;
}
.char-counter.over-limit {
  color: #e53935;
  animation: pulse 0.5s ease-in-out;
}
@keyframes pulse {
  50% {
    transform: scale(1.1);
  }
}
```

---

## 5. Toggle and Checkbox Patterns

```css
/* Toggle switch */
.toggle {
  width: 44px;
  height: 24px;
  background: #ccc;
  border-radius: 12px;
  position: relative;
  cursor: pointer;
  transition: background-color 0.2s ease-out;
}
.toggle.on {
  background: #4caf50;
}
.toggle-thumb {
  position: absolute;
  top: 2px;
  left: 2px;
  width: 20px;
  height: 20px;
  background: white;
  border-radius: 50%;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
  transition: transform 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}
.toggle.on .toggle-thumb {
  transform: translateX(20px); /* compositor-only movement */
}
```

```css
/* Animated checkmark using stroke-dashoffset */
.checkbox-icon {
  width: 18px;
  height: 18px;
}
.checkbox-icon path {
  stroke-dasharray: 20;
  stroke-dashoffset: 20;
  transition: stroke-dashoffset 0.2s ease-out;
}
.checkbox.checked .checkbox-icon path {
  stroke-dashoffset: 0; /* draws the checkmark stroke */
}
```

---

## 6. Loading and Progress Feedback

```css
/* Button loading state: spinner replaces text smoothly */
.btn-loading {
  position: relative;
  color: transparent !important; /* hide text but keep button width */
  pointer-events: none;
}
.btn-loading::after {
  content: "";
  position: absolute;
  top: 50%;
  left: 50%;
  width: 16px;
  height: 16px;
  margin: -8px 0 0 -8px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
```

```jsx
// Progressive loading states: avoid flash of spinner for fast responses
function useDelayedLoading(isLoading, delay = 200) {
  const [showLoading, setShowLoading] = useState(false);

  useEffect(() => {
    if (!isLoading) {
      setShowLoading(false);
      return;
    }
    const timer = setTimeout(() => setShowLoading(true), delay);
    return () => clearTimeout(timer);
  }, [isLoading, delay]);

  return showLoading;
}

// Usage: only show spinner if the operation takes longer than 200ms
function SaveButton({ onSave }) {
  const [isSaving, setSaving] = useState(false);
  const showSpinner = useDelayedLoading(isSaving, 200);

  return (
    <button
      className={showSpinner ? "btn-loading" : ""}
      onClick={async () => {
        setSaving(true);
        await onSave();
        setSaving(false);
      }}
    >
      Save
    </button>
  );
}
```

---

## 7. Success and Error Feedback

```css
/* Success checkmark "morph" — common pattern for form submission success */
.success-icon {
  width: 64px;
  height: 64px;
}
.success-icon .circle {
  stroke-dasharray: 166;
  stroke-dashoffset: 166;
  animation: drawCircle 0.5s ease-out forwards;
}
.success-icon .check {
  stroke-dasharray: 48;
  stroke-dashoffset: 48;
  animation: drawCheck 0.3s ease-out 0.4s forwards; /* starts after circle completes */
}
@keyframes drawCircle {
  to {
    stroke-dashoffset: 0;
  }
}
@keyframes drawCheck {
  to {
    stroke-dashoffset: 0;
  }
}
```

```jsx
// Button state machine: idle → loading → success → idle
function SubmitButton({ onSubmit }) {
  const [state, setState] = useState("idle"); // idle | loading | success | error

  const handleClick = async () => {
    setState("loading");
    try {
      await onSubmit();
      setState("success");
      setTimeout(() => setState("idle"), 1500); // revert after showing success
    } catch {
      setState("error");
      setTimeout(() => setState("idle"), 1500);
    }
  };

  return (
    <button
      className={`btn btn--${state}`}
      onClick={handleClick}
      disabled={state === "loading"}
    >
      {state === "idle" && "Submit"}
      {state === "loading" && <Spinner />}
      {state === "success" && <CheckIcon />}
      {state === "error" && <ErrorIcon />}
    </button>
  );
}
```

```css
.btn--success {
  background-color: #4caf50;
  transition: background-color 0.2s;
}
.btn--error {
  background-color: #e53935;
  animation: shake 0.4s;
}
```

---

## 8. Hover and Focus States

```css
/* Card hover: subtle lift + shadow growth, all compositor-friendly */
.card {
  transform: translateY(0);
  transition: transform 0.2s ease-out;
}
.card::after {
  content: "";
  position: absolute;
  inset: 0;
  border-radius: inherit;
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.15);
  opacity: 0;
  transition: opacity 0.2s ease-out;
  pointer-events: none;
}
.card:hover {
  transform: translateY(-4px);
}
.card:hover::after {
  opacity: 1;
}

/* Focus ring: accessible, animated entrance */
.interactive-element {
  outline: none;
  position: relative;
}
.interactive-element:focus-visible::before {
  content: "";
  position: absolute;
  inset: -3px;
  border: 2px solid #4fc3f7;
  border-radius: inherit;
  animation: focusRingIn 0.15s ease-out;
}
@keyframes focusRingIn {
  from {
    opacity: 0;
    inset: 0;
  }
  to {
    opacity: 1;
    inset: -3px;
  }
}
```

### Underline Hover Effect

```css
.link {
  position: relative;
  text-decoration: none;
}
.link::after {
  content: "";
  position: absolute;
  left: 0;
  bottom: -2px;
  width: 100%;
  height: 2px;
  background: currentColor;
  transform: scaleX(0);
  transform-origin: left;
  transition: transform 0.25s ease-out;
}
.link:hover::after {
  transform: scaleX(1);
}
```

---

## 9. Notification and Toast Patterns

```jsx
// Toast queue with auto-dismiss and stacking
function useToasts() {
  const [toasts, setToasts] = useState([]);

  const addToast = useCallback((message, type = "info", duration = 4000) => {
    const id = Date.now();
    setToasts((prev) => [...prev, { id, message, type }]);
    setTimeout(() => {
      setToasts((prev) => prev.filter((t) => t.id !== id));
    }, duration);
  }, []);

  const dismissToast = useCallback((id) => {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  }, []);

  return { toasts, addToast, dismissToast };
}

function ToastContainer({ toasts, onDismiss }) {
  return (
    <div className="toast-container" aria-live="polite">
      <AnimatePresence>
        {toasts.map((toast) => (
          <motion.div
            key={toast.id}
            layout // automatically animates position when toasts above are removed
            initial={{ opacity: 0, y: -20, scale: 0.95 }}
            animate={{ opacity: 1, y: 0, scale: 1 }}
            exit={{ opacity: 0, x: 100, transition: { duration: 0.2 } }}
            transition={{ type: "spring", stiffness: 400, damping: 30 }}
            className={`toast toast--${toast.type}`}
            onClick={() => onDismiss(toast.id)}
          >
            {toast.message}
          </motion.div>
        ))}
      </AnimatePresence>
    </div>
  );
}
```

---

## 10. Haptic-Like Visual Feedback

```javascript
// Combine visual feedback with actual haptic feedback on supported devices
function triggerFeedback(element, type = "light") {
  // Visual: scale pulse
  element.animate(
    [
      { transform: "scale(1)" },
      { transform: "scale(0.95)" },
      { transform: "scale(1)" },
    ],
    { duration: 150, easing: "ease-out" },
  );

  // Haptic: Vibration API (mobile web, limited support)
  if ("vibrate" in navigator) {
    const patterns = {
      light: 10,
      medium: 20,
      heavy: 40,
      success: [10, 50, 10],
    };
    navigator.vibrate(patterns[type] ?? 10);
  }
}

// iOS-style "tick" feedback for scrollable pickers
function useTickFeedback(currentValue, previousValue) {
  useEffect(() => {
    if (currentValue !== previousValue) {
      if ("vibrate" in navigator) navigator.vibrate(3); // very subtle tick
    }
  }, [currentValue, previousValue]);
}
```

---

## 11. Accessibility in Micro-Interactions

```css
/* Always provide a reduced-motion fallback */
@media (prefers-reduced-motion: reduce) {
  .ripple,
  .field-error-message,
  .toast {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

```jsx
// Ensure state changes are announced to screen readers, not just visual
function SaveStatus({ status }) {
  return (
    <>
      {/* Visual feedback */}
      <span
        className={`status-icon status-icon--${status}`}
        aria-hidden="true"
      />

      {/* Screen reader announcement (visually hidden but accessible) */}
      <span className="sr-only" role="status" aria-live="polite">
        {status === "saving" && "Saving changes..."}
        {status === "saved" && "Changes saved successfully"}
        {status === "error" && "Failed to save changes"}
      </span>
    </>
  );
}
```

```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

```jsx
// Don't rely on color alone for feedback (colorblind accessibility)
function ValidationIcon({ isValid }) {
  return isValid ? (
    <CheckIcon className="text-green-600" aria-label="Valid" /> // ✓ shape + color
  ) : (
    <XIcon className="text-red-600" aria-label="Invalid" />
  ); // ✗ shape + color
  // Shape conveys meaning independent of color perception
}
```

---

## 12. Good Practices

### ✅ Keep feedback proportional to the action's significance

```
Minor actions (hover, focus): subtle, fast (100-150ms)
Meaningful actions (save, submit): clear, moderate (200-300ms)
Major milestones (purchase complete, achievement): more elaborate, but still <600ms
```

### ✅ Debounce rapid re-triggers

```javascript
// ✅ Prevent animation pile-up from rapid clicking
const handleLike = useMemo(
  () =>
    debounce((id) => toggleLike(id), 150, { leading: true, trailing: false }),
  [],
);
```

### ✅ Always pair visual feedback with accessible announcements

```jsx
<div role="status" aria-live="polite" className="sr-only">
  {statusMessage}
</div>
```

---

## 13. Bad Practices

### ❌ Over-animating low-significance interactions

```css
/* ❌ Excessive animation on every checkbox click — annoying at scale */
.checkbox {
  animation: elaborateBounce 0.8s cubic-bezier(0.68, -0.55, 0.27, 1.55);
}
/* For frequently-repeated actions: keep it SHORT and SUBTLE */
```

### ❌ Blocking interaction during feedback animation

```jsx
// ❌ User can't act again until animation fully completes
<button disabled={isAnimating} onClick={handleLike}>
  {/* if isAnimating takes 600ms, user can't toggle like for 600ms — feels broken */}
</button>
// ✅ Keep feedback animations short enough that they don't meaningfully block interaction
// Or: allow interrupting/restarting the animation
```

---

## 14. Common Mistakes

### Mistake 1 — Spinner flashes for fast operations

```jsx
// ❌ Spinner shows even for 50ms operations — flickers, feels jittery
{
  isLoading && <Spinner />;
}

// ✅ Delay showing the spinner (see Section 6 useDelayedLoading)
{
  showSpinnerAfterDelay && <Spinner />;
}
```

### Mistake 2 — Toast stack without exit coordination

```jsx
// ❌ Toasts disappearing abruptly causes jarring layout shift for remaining toasts
// ✅ Use layout animation (Framer Motion's `layout` prop) so remaining
//    toasts smoothly slide into the vacated space
```

### Mistake 3 — Inconsistent timing across similar interactions

```css
/* ❌ Inconsistent: same TYPE of interaction (hover lift) uses different durations
   in different components — feels disjointed across the app */
.card-a:hover {
  transition: transform 0.2s;
}
.card-b:hover {
  transition: transform 0.35s;
}
.card-c:hover {
  transition: transform 0.15s;
}

/* ✅ Centralize timing values as design tokens */
:root {
  --duration-fast: 150ms;
  --duration-standard: 200ms;
  --duration-slow: 300ms;
  --ease-out: cubic-bezier(0.4, 0, 0.2, 1);
}
.card:hover {
  transition: transform var(--duration-standard) var(--ease-out);
}
```

---

## 15. Interview-Level Explanation

> **"How do you approach designing micro-interactions for a product? What makes them feel polished vs cheap?"**

**Strong answer:**

> "Micro-interactions have four components worth thinking about explicitly: the trigger (what initiates it), the rules (what changes and in what sequence), the feedback (what the user perceives), and the loop behavior (what happens on repeat or interruption). Most engineering effort goes into rules and feedback, but the loop behavior — what happens if the user clicks again before the animation finishes, or triggers it rapidly — is where a lot of polish is lost if not considered upfront.
>
> Timing consistency matters more than people expect. I keep a small set of design tokens — fast (150ms), standard (200ms), slow (300ms) — and map every interaction type to one of them consistently across the app. If hover-lift takes 200ms on one card component and 350ms on another, the inconsistency reads as 'unpolished' even if users can't articulate why. The convention I follow is ease-out for things entering or appearing (fast start gives immediate feedback, slow settle feels natural), ease-in for things leaving, and ease-in-out for repositioning.
>
> The other discipline is matching feedback weight to action significance. A checkbox toggle that happens dozens of times per session should have subtle, fast feedback — anything elaborate becomes annoying at that frequency. A purchase confirmation, which happens rarely and matters a lot, can have more elaborate feedback because the user isn't fatigued by repetition.
>
> Engineering-wise, the critical constraint is staying on the compositor thread — animating transform and opacity rather than layout-triggering properties — so feedback stays smooth even under load. I also delay showing loading spinners by 150-200ms; if an operation completes faster than that, showing and immediately hiding a spinner creates a distracting flash rather than useful feedback.
>
> Accessibility is non-negotiable: every visual feedback needs an accessible equivalent. A status icon that turns green needs an aria-live region announcing 'saved successfully' for screen reader users, and shouldn't rely on color alone — pairing a checkmark shape with the green color so colorblind users get the same information. And everything needs a prefers-reduced-motion fallback for users with vestibular sensitivities."

---

## 16. Exercises

### Exercise 1 — Design a "copy to clipboard" interaction

Design the complete micro-interaction for a "copy link" button: states, timing, accessibility, and edge case handling (rapid clicks, clipboard API failure).

<details>
<summary>Solution</summary>

```jsx
function CopyButton({ text }) {
  const [status, setStatus] = useState("idle"); // idle | copied | error
  const timeoutRef = useRef();

  const handleCopy = async () => {
    // Clear any pending revert from a previous click
    clearTimeout(timeoutRef.current);

    try {
      await navigator.clipboard.writeText(text);
      setStatus("copied");
    } catch {
      // Fallback for browsers without Clipboard API permission
      try {
        const textarea = document.createElement("textarea");
        textarea.value = text;
        textarea.style.position = "fixed";
        textarea.style.opacity = "0";
        document.body.appendChild(textarea);
        textarea.select();
        document.execCommand("copy");
        document.body.removeChild(textarea);
        setStatus("copied");
      } catch {
        setStatus("error");
      }
    }

    timeoutRef.current = setTimeout(() => setStatus("idle"), 2000);
  };

  useEffect(() => () => clearTimeout(timeoutRef.current), []);

  return (
    <button
      className={`copy-btn copy-btn--${status}`}
      onClick={handleCopy}
      aria-label={status === "copied" ? "Copied to clipboard" : "Copy link"}
    >
      <span className="copy-btn__icon" aria-hidden="true">
        {status === "idle" && <LinkIcon />}
        {status === "copied" && <CheckIcon />}
        {status === "error" && <ErrorIcon />}
      </span>
      <span className="copy-btn__label">
        {status === "idle" && "Copy link"}
        {status === "copied" && "Copied!"}
        {status === "error" && "Failed to copy"}
      </span>

      {/* Screen reader announcement */}
      <span className="sr-only" role="status" aria-live="polite">
        {status === "copied" && "Link copied to clipboard"}
        {status === "error" && "Failed to copy link"}
      </span>
    </button>
  );
}
```

```css
.copy-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: background-color 0.15s ease-out;
}

.copy-btn__icon {
  display: inline-flex;
  transition: transform 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

.copy-btn--copied {
  background-color: #e8f5e9;
}
.copy-btn--copied .copy-btn__icon {
  animation: iconPop 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}
@keyframes iconPop {
  0% {
    transform: scale(0.5);
    opacity: 0;
  }
  60% {
    transform: scale(1.15);
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

.copy-btn--error {
  background-color: #ffebee;
  animation: shake 0.4s ease-in-out;
}

@media (prefers-reduced-motion: reduce) {
  .copy-btn__icon,
  .copy-btn--copied .copy-btn__icon,
  .copy-btn--error {
    animation: none !important;
  }
}

/* Design decisions:
   - Idle → Copied transition: 200-300ms total (icon pop draws attention briefly)
   - Auto-revert after 2s: gives enough time to register without lingering
   - Rapid re-click: clearTimeout prevents premature revert if clicked again
   - Fallback for older browsers / permission-denied Clipboard API
   - Accessible: aria-label updates, aria-live region for screen readers
   - Reduced motion: icon pop disabled, button still changes state via color/text
*/
```

</details>

---

## 🔗 Related Topics

- [`animations/01-css-animations.md`](./01-css-animations.md) — CSS animation fundamentals
- [`animations/03-compositor-animations.md`](./03-compositor-animations.md) — Performance-safe animation
- [`patterns/`](../patterns/) — Broader UI design patterns
- [`accessibility`](../docs/glossary.md) — Accessibility terminology reference

---

<div align="center">

**`animations/` section complete!** 🎉

All 4 animations files done:
`01-css-animations.md` · `02-javascript-animations.md` · `03-compositor-animations.md` · **`04-micro-interactions.md`** ✓

**Next section:** [`patterns/`](../patterns/) →

</div>
