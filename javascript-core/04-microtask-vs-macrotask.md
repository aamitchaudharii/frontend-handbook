# 04 — Microtasks vs Macrotasks

> **"The difference between a microtask and a macrotask is not just about timing — it's about priority, rendering, and whether the browser gets a chance to breathe between your callbacks."**

Most developers know that Promises run before setTimeout. Very few know _why_ — or all the places that distinction matters in production code. This document covers the complete task queue model: what goes where, the exact processing order, how rendering fits in, and the real-world bugs this model creates.

---

## 📚 Table of Contents

1. [The Two Queues — Overview](#1-the-two-queues--overview)
2. [What Is a Macrotask?](#2-what-is-a-macrotask)
3. [What Is a Microtask?](#3-what-is-a-microtask)
4. [The Processing Algorithm — Exact Order](#4-the-processing-algorithm--exact-order)
5. [The Rendering Checkpoint](#5-the-rendering-checkpoint)
6. [Microtask Checkpoint — When It Runs](#6-microtask-checkpoint--when-it-runs)
7. [Queue Priority Visualization](#7-queue-priority-visualization)
8. [Real Execution Traces](#8-real-execution-traces)
9. [The requestAnimationFrame Queue](#9-the-requestanimationframe-queue)
10. [The requestIdleCallback Queue](#10-the-requestidlecallback-queue)
11. [queueMicrotask — Direct API](#11-queuemicrotask--direct-api)
12. [Practical Patterns and Use Cases](#12-practical-patterns-and-use-cases)
13. [Starvation — When Microtasks Go Wrong](#13-starvation--when-microtasks-go-wrong)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. The Two Queues — Overview

The JavaScript runtime maintains multiple queues for deferred work. The two most important are the **microtask queue** and the **macrotask queue** (also called the task queue).

```
┌─────────────────────────────────────────────────────────────────────┐
│                      JAVASCRIPT RUNTIME                              │
│                                                                       │
│  ┌─────────────┐                                                     │
│  │ Call Stack  │ ← currently executing                               │
│  └──────┬──────┘                                                     │
│         │ empties                                                     │
│         ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              MICROTASK QUEUE  (high priority)                │    │
│  │  Promises · queueMicrotask · MutationObserver               │    │
│  │  ──────────────────────────────────────────────             │    │
│  │  Drained COMPLETELY before anything else runs               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │ after microtask queue is empty                             │
│         ▼                                                             │
│  ┌──────────────────────┐                                            │
│  │  Rendering Checkpoint│ ← browser may paint a frame here           │
│  │  (rAF callbacks run) │                                            │
│  └──────────────────────┘                                            │
│         │                                                             │
│         ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              MACROTASK QUEUE  (task queue)                   │    │
│  │  setTimeout · setInterval · I/O · DOM events · MessagePort  │    │
│  │  ──────────────────────────────────────────────             │    │
│  │  ONE task dequeued per event loop iteration                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

The single most important rule:

> **After every macrotask, the entire microtask queue is drained before the next macrotask runs or the browser renders.**

---

## 2. What Is a Macrotask?

A macrotask (officially called a **task** in the HTML spec) is a unit of work that the browser schedules. The event loop processes **one macrotask per iteration**, then checks for rendering and drains microtasks.

### What Goes in the Macrotask Queue

```javascript
// setTimeout — callback enqueued as macrotask after delay
setTimeout(() => console.log("macrotask"), 0);

// setInterval — repeated macrotask
setInterval(() => console.log("interval"), 1000);

// DOM event callbacks
button.addEventListener("click", () => console.log("click")); // macrotask on click

// MessageChannel — programmatic macrotask scheduling
const { port1, port2 } = new MessageChannel();
port1.onmessage = () => console.log("message");
port2.postMessage(""); // enqueues macrotask

// fetch/XHR completion callbacks
fetch("/api").then(/* this .then is a microtask, but the fetch completion
                      that triggers it is a macrotask-level operation */);

// Script loading
// <script src="..."> — parsing and executing a script is a macrotask

// setImmediate (Node.js only)
setImmediate(() => console.log("immediate")); // Node.js-specific
```

### Key Characteristics of Macrotasks

| Property        | Behavior                                         |
| --------------- | ------------------------------------------------ |
| Processing rate | ONE per event loop iteration                     |
| Rendering       | Browser may render between macrotasks            |
| Priority        | Lower than microtasks                            |
| Starvation risk | Unlikely (browser enforces one-per-loop)         |
| Use case        | Timer delays, I/O callbacks, yielding to browser |

### `setTimeout(fn, 0)` Is NOT Zero Delay

```javascript
// Common misconception:
setTimeout(fn, 0); // "runs immediately after current code"

// Reality:
// 1. Even with delay=0, browsers have a minimum clamp of ~4ms
//    for nested timeouts (HTML spec: after 5 levels of nesting,
//    minimum interval is 4ms)
// 2. Runs AFTER all pending microtasks
// 3. Runs AFTER the browser may render a frame
// 4. Subject to background tab throttling (1000ms+ in hidden tabs)
```

---

## 3. What Is a Microtask?

A microtask is a smaller, higher-priority unit of work. The microtask queue is drained **completely** after every macrotask and after every microtask checkpoint (which can happen at other points too — covered in Section 6).

### What Goes in the Microtask Queue

```javascript
// Promise .then / .catch / .finally callbacks
Promise.resolve().then(() => console.log("microtask"));
Promise.reject(err).catch(() => console.log("catch microtask"));
Promise.resolve().finally(() => console.log("finally microtask"));

// async/await continuations (they desugar to Promise callbacks)
async function foo() {
  await something(); // everything after await is a microtask callback
  console.log("after await — microtask");
}

// queueMicrotask — direct API
queueMicrotask(() => console.log("direct microtask"));

// MutationObserver callbacks
const mo = new MutationObserver(() => console.log("mutation microtask"));
mo.observe(el, { childList: true });

// Promise-based APIs (their resolution callbacks are microtasks)
// fetch().then() — the .then callback is a microtask
// (the network response itself is a macrotask trigger, but
//  the .then callback runs in microtask queue)
```

### Key Characteristics of Microtasks

| Property        | Behavior                                                |
| --------------- | ------------------------------------------------------- |
| Processing rate | ALL of them, every checkpoint                           |
| Rendering       | Never between microtasks — only after full drain        |
| Priority        | Higher than macrotasks                                  |
| Starvation risk | HIGH — can starve rendering and macrotasks              |
| Use case        | Post-sync work, state sync, batch updates before render |

### The Critical Difference from Macrotasks

```javascript
// One macrotask runs, then entire microtask queue drains, then next macrotask
// This means: queuing a microtask FROM a microtask still runs before
// any macrotask or render

Promise.resolve().then(() => {
  console.log("microtask 1");
  // Queue another microtask from within a microtask
  Promise.resolve().then(() => {
    console.log("microtask 2 — queued inside microtask 1");
    // This STILL runs before any setTimeout or render
  });
});

setTimeout(() => console.log("macrotask"));

// Output:
// microtask 1
// microtask 2 — queued inside microtask 1
// macrotask
```

---

## 4. The Processing Algorithm — Exact Order

Here is the complete event loop processing order (simplified from the HTML Living Standard):

```
EVENT LOOP ITERATION:
─────────────────────────────────────────────────────────────

1. TASK STEP
   └── Dequeue ONE task from the macrotask queue
       (or run the initial script if first iteration)
       └── Execute it completely on the call stack

2. MICROTASK CHECKPOINT
   └── While microtask queue is NOT empty:
         └── Dequeue the oldest microtask
             └── Execute it
             └── (it may enqueue more microtasks — those run too)
       Repeat until queue is completely empty

3. RENDERING STEP (if a rendering opportunity exists)
   ├── Run resize observers
   ├── Run scroll observers
   ├── Run animation frame callbacks (requestAnimationFrame)
   ├── Perform style recalculation
   ├── Perform layout
   ├── Perform paint
   └── Composite to screen

4. IDLE STEP (if idle time remains in the frame)
   └── Run requestIdleCallback callbacks

5. GO TO STEP 1

─────────────────────────────────────────────────────────────
```

**The rendering step only happens if the browser decides it's time.** At 60fps, the browser targets a render every ~16.67ms. It won't render on every event loop iteration — only when the display refresh rate demands it.

---

## 5. The Rendering Checkpoint

The rendering step sits **between** the microtask drain and the next macrotask. This has major implications:

```
MacroTask → [drain all microtasks] → [maybe render] → MacroTask → ...
```

### What This Means for DOM Updates

```javascript
// DOM mutations inside a macrotask:
button.addEventListener("click", () => {
  // This is a macrotask (click event callback)

  document.body.style.background = "red"; // mutation #1
  document.body.style.background = "blue"; // mutation #2
  document.body.style.background = "green"; // mutation #3

  // The browser doesn't render after each line.
  // It renders AFTER this entire macrotask completes
  // and microtasks drain.
  // User will only ever see the FINAL state: green background.
  // Red and blue are never painted.
});
```

This is actually beneficial — the browser batches multiple style changes and renders once, avoiding intermediate visual flicker.

### What This Means for Animations

```javascript
// ❌ Using setTimeout for animation:
function animateBad(progress = 0) {
  element.style.transform = `translateX(${progress}px)`;
  if (progress < 300) {
    setTimeout(() => animateBad(progress + 1), 0);
    // setTimeout is a macrotask — render MAY happen between frames
    // But timing is not synced to display refresh — results in uneven animation
  }
}

// ✅ Using requestAnimationFrame:
function animateGood(progress = 0) {
  element.style.transform = `translateX(${progress}px)`;
  if (progress < 300) {
    requestAnimationFrame(() => animateGood(progress + 1));
    // RAF callback runs AT the rendering checkpoint
    // Synced to display refresh rate — smooth 60fps
  }
}
```

---

## 6. Microtask Checkpoint — When It Runs

The microtask checkpoint runs more often than just after each macrotask. It runs after:

1. **After each macrotask** (the main case)
2. **After each script evaluation** (after a `<script>` tag finishes)
3. **After each task** (some async API completions trigger it)
4. **When explicitly called** by the browser in certain spec algorithms

```javascript
// Example: microtask checkpoint after script evaluation

// In HTML: when an inline <script> finishes, microtasks run before
// the browser continues parsing HTML or running the next script

// <script>
//   Promise.resolve().then(() => console.log('microtask after script'));
//   console.log('script end');
// </script>
// <script>
//   console.log('next script');
// </script>

// Output:
// script end
// microtask after script  ← runs before next <script> executes
// next script
```

---

## 7. Queue Priority Visualization

```
Priority (highest → lowest):

┌─────────────────────────────────────────────────────────────────┐
│  1. SYNCHRONOUS CODE (call stack)                               │
│     Current executing code — runs to completion first           │
├─────────────────────────────────────────────────────────────────┤
│  2. MICROTASK QUEUE                                             │
│     Promises, queueMicrotask, MutationObserver                 │
│     ALL drained before anything below runs                      │
├─────────────────────────────────────────────────────────────────┤
│  3. ANIMATION FRAME CALLBACKS (requestAnimationFrame)           │
│     Runs at rendering checkpoint — before browser paints        │
├─────────────────────────────────────────────────────────────────┤
│  4. MACROTASK QUEUE (Task Queue)                                │
│     setTimeout, setInterval, DOM events, I/O                   │
│     ONE per event loop iteration                                │
├─────────────────────────────────────────────────────────────────┤
│  5. IDLE CALLBACKS (requestIdleCallback)                        │
│     Only runs when browser has spare time                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Real Execution Traces

### Trace 1 — The Classic

```javascript
console.log("1"); // sync

setTimeout(() => console.log("2"), 0); // macrotask

Promise.resolve()
  .then(() => console.log("3")) // microtask
  .then(() => console.log("4")); // microtask (chained)

console.log("5"); // sync
```

```
EXECUTION TRACE:
─────────────────────────────────────────────────────
Synchronous phase:
  console.log('1')  → prints 1
  setTimeout(fn,0)  → fn queued as macrotask
  Promise.resolve().then(→3) → microtask queued: [→3]
  console.log('5')  → prints 5

Call stack empty. Run microtask checkpoint:
  Microtask queue: [→3]
  Run →3: prints 3
  .then(→4) is now queued: [→4]

  Microtask queue: [→4]
  Run →4: prints 4

  Microtask queue: [] ← empty. Stop.

Rendering checkpoint (browser may render here)

Next macrotask:
  Run setTimeout fn: prints 2

─────────────────────────────────────────────────────
OUTPUT: 1, 5, 3, 4, 2
```

### Trace 2 — Mixed Async

```javascript
console.log("start");

setTimeout(() => {
  console.log("timeout 1");
  Promise.resolve().then(() => console.log("promise inside timeout"));
}, 0);

setTimeout(() => console.log("timeout 2"), 0);

Promise.resolve()
  .then(() => console.log("promise 1"))
  .then(() => console.log("promise 2"));

console.log("end");
```

```
EXECUTION TRACE:
─────────────────────────────────────────────────────
Synchronous:
  'start' → prints
  setTimeout(t1, 0) → macrotask queue: [t1]
  setTimeout(t2, 0) → macrotask queue: [t1, t2]
  Promise → microtask queue: [p1]
  'end' → prints

Microtask checkpoint:
  Run p1 → prints 'promise 1', queues p2
  Run p2 → prints 'promise 2'
  Queue empty.

Rendering checkpoint.

Macrotask: dequeue t1
  'timeout 1' → prints
  Promise.resolve().then → microtask queue: [p-inside]
  t1 finishes.

  Microtask checkpoint (AFTER t1 macrotask):
  Run p-inside → prints 'promise inside timeout'
  Queue empty.

Rendering checkpoint.

Macrotask: dequeue t2
  'timeout 2' → prints

─────────────────────────────────────────────────────
OUTPUT: start, end, promise 1, promise 2,
        timeout 1, promise inside timeout, timeout 2
```

The key insight: the microtask queued **inside** `timeout 1` runs before `timeout 2` — not after both timeouts.

### Trace 3 — async/await

```javascript
async function asyncFn() {
  console.log("async start");
  await Promise.resolve();
  console.log("after await 1");
  await Promise.resolve();
  console.log("after await 2");
}

console.log("sync 1");
asyncFn();
console.log("sync 2");
```

```
EXECUTION TRACE:
─────────────────────────────────────────────────────
Synchronous:
  'sync 1' → prints
  asyncFn() called:
    'async start' → prints
    hits first `await Promise.resolve()`
    → suspends asyncFn, queues continuation as microtask: [resume-1]
    → asyncFn returns (its Promise — not awaited here)
  'sync 2' → prints

Microtask checkpoint:
  Run resume-1:
    'after await 1' → prints
    hits second `await Promise.resolve()`
    → suspends again, queues continuation: [resume-2]

  Run resume-2:
    'after await 2' → prints

─────────────────────────────────────────────────────
OUTPUT: sync 1, async start, sync 2, after await 1, after await 2
```

Each `await` inserts a microtask checkpoint. Execution after each `await` is a separate microtask.

### Trace 4 — The Tricky `return Promise.resolve()`

```javascript
Promise.resolve()
  .then(() => {
    console.log("A");
    return Promise.resolve("B"); // returning a Promise from .then
  })
  .then((val) => console.log(val)); // when does this run?

Promise.resolve()
  .then(() => console.log("C"))
  .then(() => console.log("D"));
```

```
EXECUTION TRACE:
─────────────────────────────────────────────────────
Initial microtask queue: [A-handler, C-handler]

Run A-handler:
  prints 'A'
  returns Promise.resolve('B')
  → When a .then handler returns a thenable (Promise),
    the spec requires a PromiseResolveThenableJob to run
  → This adds 2 extra microtask ticks before B-handler is called
  Queue: [C-handler, PromiseResolveThenableJob]

Run C-handler:
  prints 'C', queues D-handler
  Queue: [PromiseResolveThenableJob, D-handler]

Run PromiseResolveThenableJob:
  (resolves the returned promise, queues B-handler)
  Queue: [D-handler, B-handler]

Run D-handler:
  prints 'D'
  Queue: [B-handler]

Run B-handler:
  prints 'B'

─────────────────────────────────────────────────────
OUTPUT: A, C, D, B
```

Returning `Promise.resolve()` from a `.then` handler introduces **2 extra microtask ticks** compared to returning a plain value. This is a notorious interview question.

---

## 9. The requestAnimationFrame Queue

`requestAnimationFrame` (rAF) callbacks live in a separate queue — the **animation frame callbacks queue**. They run at the rendering checkpoint, after microtasks drain and before the browser paints.

```
Event loop iteration timeline:

[Macrotask] → [Microtasks] → [rAF callbacks] → [Layout/Paint/Composite] → [Idle]
                                    ↑
                         This is the rendering checkpoint
```

### Key Properties of rAF

```javascript
// rAF callback receives a DOMHighResTimeStamp
requestAnimationFrame(function (timestamp) {
  console.log(timestamp); // ms since page load, e.g. 1234.56

  // This callback runs BEFORE the browser paints — so mutations
  // here are included in the CURRENT frame's paint
  element.style.transform = "translateX(100px)"; // painted this frame
});
```

### rAF vs setTimeout for Animation

```
setTimeout(fn, 0):
  - Macrotask — no guaranteed sync with display
  - Subject to ~4ms minimum clamp
  - May run multiple times per frame or skip frames
  - Background throttled
  - NOT synced to vsync

requestAnimationFrame(fn):
  - Animation checkpoint — synced to display refresh rate
  - Fires ~60 times/second on 60Hz displays, 120 on 120Hz
  - Only once per frame (not twice or zero)
  - Paused in background tabs (saves battery)
  - Guaranteed to run before paint — no visual tearing
```

### rAF Callback Queue Processing

**Important:** ALL rAF callbacks queued before the rendering checkpoint run in that frame. Callbacks queued from within a rAF callback run in the **next** frame.

```javascript
// rAF callback A queues another rAF
requestAnimationFrame(function A() {
  console.log("A — frame 1");
  requestAnimationFrame(function B() {
    console.log("B — frame 2"); // runs next frame, not this one
  });
});

requestAnimationFrame(function C() {
  console.log("C — frame 1"); // runs same frame as A
});

// Frame 1: A, C
// Frame 2: B
```

---

## 10. The requestIdleCallback Queue

`requestIdleCallback` runs during **idle periods** — time the browser has left over after completing a frame's required work.

```
Frame timeline (16.67ms at 60fps):

[Input] [JS/Macrotask] [Microtasks] [rAF] [Layout] [Paint] [Composite]
                                                                         ↑
                                                              Idle time starts here
                                                              requestIdleCallback runs
```

### Properties

```javascript
requestIdleCallback(
  function (deadline) {
    // deadline.timeRemaining() — ms left in idle period (0-50ms)
    // deadline.didTimeout      — true if called due to timeout, not idle time

    while (deadline.timeRemaining() > 0 && tasks.length > 0) {
      processTask(tasks.shift());
    }

    if (tasks.length > 0) {
      requestIdleCallback(continueProcessing); // schedule more if unfinished
    }
  },
  { timeout: 2000 },
); // force run within 2 seconds even if not idle
```

### When to Use rIC

- Analytics event batching
- Non-urgent prefetching
- Background cleanup tasks
- Lazy-loading non-critical resources

**Never use for:** User-visible updates, animations, or anything time-sensitive. Idle time is not guaranteed and the callback may not run for seconds.

---

## 11. `queueMicrotask` — Direct API

`queueMicrotask` lets you schedule a microtask directly without creating a Promise. It's cleaner and slightly faster.

```javascript
// ✅ Direct microtask scheduling
queueMicrotask(() => {
  console.log("this is a microtask");
});

// vs. Promise workaround (achieves same result, but creates Promise object)
Promise.resolve().then(() => {
  console.log("also a microtask but allocates a Promise");
});
```

### Use Case — Deferred State Sync

```javascript
class ReactiveStore {
  constructor(state) {
    this._state = state;
    this._listeners = new Set();
    this._dirty = false;
  }

  set(key, value) {
    this._state[key] = value;

    if (!this._dirty) {
      this._dirty = true;
      // Defer notification until after ALL synchronous state changes
      // This way 5 rapid .set() calls only trigger ONE notification
      queueMicrotask(() => {
        this._dirty = false;
        this._notify();
      });
    }
  }

  _notify() {
    this._listeners.forEach((fn) => fn(this._state));
  }

  subscribe(fn) {
    this._listeners.add(fn);
    return () => this._listeners.delete(fn);
  }
}

// Usage:
const store = new ReactiveStore({ count: 0 });
store.subscribe((state) => console.log("updated:", state.count));

store.set("count", 1); // doesn't notify yet
store.set("count", 2); // doesn't notify yet
store.set("count", 3); // doesn't notify yet
// After sync code: ONE notification with final state: 3
```

---

## 12. Practical Patterns and Use Cases

### Pattern 1 — Post-Render Read

Sometimes you need to read a DOM measurement **after** the browser has rendered. Use a double-rAF:

```javascript
// Single rAF: runs before paint — layout may not be complete for NEW elements
requestAnimationFrame(() => {
  // Browser hasn't painted yet — may not have latest layout for new content
  const height = newElement.offsetHeight; // may be 0 for newly added elements
});

// Double rAF: first rAF runs before paint, second runs after
requestAnimationFrame(() => {
  requestAnimationFrame(() => {
    // Now we're in the NEXT frame — previous frame is fully rendered and laid out
    const height = newElement.offsetHeight; // accurate
  });
});
```

### Pattern 2 — Yield to Main Thread

Break up long tasks by yielding via setTimeout (macrotask) to let the browser render and handle input:

```javascript
async function processLargeArray(items) {
  const CHUNK = 100;

  for (let i = 0; i < items.length; i += CHUNK) {
    const chunk = items.slice(i, i + CHUNK);
    processChunk(chunk);

    // Yield to browser between chunks
    // setTimeout makes this a macrotask boundary → browser can render
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}
```

### Pattern 3 — Immediate vs Deferred State Update

```javascript
class Component {
  // Immediate: update + notify right now (synchronous callers see it)
  setStateSync(updates) {
    Object.assign(this.state, updates);
    this.render();
  }

  // Deferred: batch multiple updates into one render (microtask)
  setState(updates) {
    Object.assign(this.state, updates);
    if (!this._renderScheduled) {
      this._renderScheduled = true;
      queueMicrotask(() => {
        this._renderScheduled = false;
        this.render();
      });
    }
  }
}
```

### Pattern 4 — Measuring DOM After Script Changes

```javascript
// You want to measure element size after you've added content
function addAndMeasure(container, content) {
  container.textContent = content; // DOM write

  // Immediately reading would force synchronous layout (layout thrashing)
  // Instead, measure in rAF — after browser has processed the change
  requestAnimationFrame(() => {
    const height = container.offsetHeight; // accurate, no forced layout
    adjustLayout(height);
  });
}
```

---

## 13. Starvation — When Microtasks Go Wrong

Because the microtask queue is drained **completely** before anything else runs, an infinite or very long microtask chain can **starve** the browser — preventing rendering, user input, and macrotask execution indefinitely.

### Infinite Microtask Chain

```javascript
// ❌ NEVER DO THIS — starves the event loop completely
function infiniteMicrotasks() {
  Promise.resolve().then(infiniteMicrotasks);
}
infiniteMicrotasks();

// What happens:
// 1. Promise resolves → calls infiniteMicrotasks
// 2. infiniteMicrotasks queues ANOTHER microtask
// 3. That microtask queues ANOTHER microtask
// 4. ...forever
// 5. Microtask queue NEVER empties
// 6. Browser NEVER gets to the rendering checkpoint
// 7. Page is completely frozen
```

### Long Microtask Chain (practical version)

```javascript
// ❌ Synchronously resolving 100,000 promises
// This keeps the main thread busy for hundreds of milliseconds
async function processAll(items) {
  for (const item of items) {
    await Promise.resolve(process(item));
    // Each await = microtask checkpoint, but we queue the NEXT
    // continuation immediately — browser may never get to render
  }
}

// ✅ Periodically yield to browser with a macrotask break
async function processAll(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 100 === 0) {
      // Yield to browser: allow render + input handling
      await new Promise((resolve) => setTimeout(resolve, 0));
    }
  }
}
```

### Detecting Microtask Starvation

```javascript
// Monitor if frames are being skipped
let lastFrame = performance.now();

requestAnimationFrame(function checkStarvation(now) {
  const gap = now - lastFrame;
  if (gap > 50) {
    // frame took > 50ms (should be ~16ms)
    console.warn(`Potential starvation: ${gap.toFixed(1)}ms between frames`);
  }
  lastFrame = now;
  requestAnimationFrame(checkStarvation);
});
```

---

## 14. Good Practices

### ✅ Use `queueMicrotask` for post-sync deferred work (not Promise)

```javascript
// ✅ Cleaner, slightly faster, no Promise allocation
queueMicrotask(() => notifySubscribers());

// vs.
Promise.resolve().then(() => notifySubscribers()); // creates unnecessary Promise object
```

### ✅ Use `requestAnimationFrame` for ALL visual updates

```javascript
// ✅ Any DOM change that will be visually rendered
function updateProgress(value) {
  requestAnimationFrame(() => {
    progressBar.style.transform = `scaleX(${value})`;
  });
}
```

### ✅ Yield to the main thread in long operations

```javascript
// ✅ Use setTimeout(0) as a yield mechanism for long tasks
const yieldToMain = () => new Promise((r) => setTimeout(r, 0));

async function longOperation(items) {
  for (let i = 0; i < items.length; i++) {
    doWork(items[i]);
    if (i % 50 === 0) await yieldToMain(); // periodic yield
  }
}
```

### ✅ Know which queue your code goes into

```javascript
// Be intentional about scheduling:
queueMicrotask(fn); // after current sync code, before render
requestAnimationFrame(fn); // at render time, before paint
setTimeout(fn, 0); // after render, in next event loop tick
requestIdleCallback(fn); // when browser is idle
```

---

## 15. Bad Practices

### ❌ Recursive microtask scheduling

```javascript
// ❌ Even if not infinite — long chains delay rendering
async function deepChain(n) {
  if (n === 0) return;
  await Promise.resolve();
  return deepChain(n - 1); // 10,000 microtask checkpoints before rendering
}
deepChain(10000);
```

### ❌ Using setTimeout for state batching (use queueMicrotask instead)

```javascript
// ❌ setTimeout batching delays rendering unnecessarily
setTimeout(() => batchedUpdate(), 0); // adds a full macrotask delay + potential render

// ✅ queueMicrotask batching happens before render — better UX
queueMicrotask(() => batchedUpdate());
```

### ❌ Assuming Promise callbacks run "immediately"

```javascript
let result;

Promise.resolve(42).then((val) => {
  result = val;
});

console.log(result); // ❌ undefined — .then hasn't run yet
// .then is a microtask — runs AFTER synchronous code
```

### ❌ DOM reads inside rAF to measure, then write in same rAF

```javascript
// ❌ Still causes layout thrashing within the rAF callback
requestAnimationFrame(() => {
  elements.forEach((el) => {
    const h = el.offsetHeight; // READ: forces layout
    el.style.height = h + 10 + "px"; // WRITE: invalidates
  });
});

// ✅ Separate reads and writes even inside rAF
requestAnimationFrame(() => {
  const heights = elements.map((el) => el.offsetHeight); // all reads
  elements.forEach((el, i) => (el.style.height = heights[i] + 10 + "px")); // all writes
});
```

---

## 16. Common Mistakes

### Mistake 1 — Treating async/await as truly synchronous

```javascript
async function getData() {
  const data = await fetch("/api").then((r) => r.json());
  return data;
}

const result = getData();
console.log(result); // ❌ Promise { <pending> } — not the data
// await does NOT block the caller — it suspends the async function
// and returns a Promise to the caller immediately
```

### Mistake 2 — Assuming Promise.all resolves in a single microtask

```javascript
// Promise.all with N promises may take N+ microtask ticks to resolve
// depending on when each promise resolves
const [a, b, c] = await Promise.all([p1, p2, p3]);
// All three must resolve before this line runs
// If p1 resolves in 1 tick, p2 in 3 ticks, p3 in 2 ticks:
// Promise.all resolves after 3 microtask ticks (waits for slowest)
```

### Mistake 3 — Nesting rAF inside a microtask

```javascript
// ❌ Confusing execution order
Promise.resolve().then(() => {
  requestAnimationFrame(() => {
    console.log("inside rAF inside microtask");
    // This runs in the NEXT rendering checkpoint, not this one
    // and definitely not during the microtask phase
  });
  console.log("after rAF registration");
});
// Output: 'after rAF registration', then (next frame) 'inside rAF inside microtask'
```

### Mistake 4 — Using rAF for non-visual work

```javascript
// ❌ rAF is for VISUAL updates only
// Using it for non-visual work wastes the rendering budget
requestAnimationFrame(() => {
  processAnalytics(); // this has nothing to do with rendering
  saveToLocalStorage(); // this should be in rIC or setTimeout
});

// ✅ rIC for non-urgent non-visual work
requestIdleCallback(() => {
  processAnalytics();
  saveToLocalStorage();
});
```

---

## 17. Interview-Level Explanation

> **"What's the difference between microtasks and macrotasks?"**

**Strong answer:**

> "Both are mechanisms for deferring code execution, but they have fundamentally different priority and processing rules.
>
> A macrotask — like a setTimeout callback, a DOM event handler, or an I/O callback — is dequeued one at a time per event loop iteration. After a macrotask finishes, the browser gets a chance to render before processing the next one.
>
> A microtask — like a Promise .then callback or a queueMicrotask call — is processed in the microtask checkpoint, which runs after every macrotask and drains the ENTIRE queue before anything else can happen. If a microtask queues another microtask, that also runs before any macrotask or rendering. The entire queue must be empty before control returns to the event loop.
>
> This is why Promises always resolve before setTimeout — Promises use microtasks, setTimeout uses macrotasks, and microtasks have higher priority.
>
> The rendering step — where requestAnimationFrame callbacks run and the browser actually paints — sits between the microtask drain and the next macrotask. This means microtask-heavy code can delay rendering, while macrotask-heavy code gives the browser a chance to breathe between each task.
>
> A practical implication: for state batching that should happen before the next render, use queueMicrotask. For yielding the main thread to allow rendering, use setTimeout or requestAnimationFrame."

---

## 18. Exercises

### Exercise 1 — Predict the output

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve()
  .then(() => {
    console.log("3");
    setTimeout(() => console.log("4"), 0);
    return Promise.resolve();
  })
  .then(() => console.log("5"));

queueMicrotask(() => console.log("6"));

setTimeout(() => {
  console.log("7");
  Promise.resolve().then(() => console.log("8"));
}, 0);

console.log("9");
```

<details>
<summary>Answer</summary>

```
Synchronous:
  1 → prints
  setTimeout(→2) → macrotask queue: [→2]
  Promise.then(→3) → microtask queue: [→3]
  queueMicrotask(→6) → microtask queue: [→3, →6]
  setTimeout(→7) → macrotask queue: [→2, →7]
  9 → prints

Microtask checkpoint:
  →3 runs: prints '3'
        setTimeout(→4) → macrotask queue: [→2, →7, →4]
        returns Promise.resolve() → 2 extra ticks for PromiseResolveThenableJob
        queue: [→6, PromiseResolveThenableJob]
  →6 runs: prints '6'
        queue: [PromiseResolveThenableJob]
  PromiseResolveThenableJob runs (resolves returned promise, queues →5)
        queue: [→5... wait, need one more tick]
        Actually this adds the then-handler: queue: [resolve-microtask]
  resolve-microtask runs → now queues →5: [→5]
  →5 runs: prints '5'
  Queue empty.

Macrotask: →2 → prints '2'
  Microtask checkpoint: (empty)

Macrotask: →7 → prints '7'
  Promise.resolve().then(→8) → microtask: [→8]
  Microtask checkpoint: →8 → prints '8'

Macrotask: →4 → prints '4'

OUTPUT: 1, 9, 3, 6, 5, 2, 7, 8, 4
```

</details>

---

### Exercise 2 — Fix the starvation

```javascript
// ❌ This UI update never appears because of microtask starvation
// Fix it so the loading indicator appears before processing starts

async function processItems(items) {
  showLoadingIndicator(); // DOM change — but user never sees it

  for (const item of items) {
    await processItemAsync(item); // 10,000 microtask ticks
  }

  hideLoadingIndicator();
}
```

<details>
<summary>Solution</summary>

```javascript
async function processItems(items) {
  showLoadingIndicator();

  // ✅ Yield to browser via macrotask — allows render checkpoint
  // so loading indicator is actually painted before processing
  await new Promise((resolve) => requestAnimationFrame(resolve));
  // Or: await new Promise(resolve => setTimeout(resolve, 0));

  for (let i = 0; i < items.length; i++) {
    await processItemAsync(items[i]);

    // Periodically yield to keep UI responsive
    if (i % 100 === 0) {
      await new Promise((resolve) => setTimeout(resolve, 0));
    }
  }

  hideLoadingIndicator();
}
```

</details>

---

### Exercise 3 — Queue identification

For each of the following, identify which queue the callback ends up in:

```javascript
a) setTimeout(fn, 0)
b) Promise.resolve().then(fn)
c) requestAnimationFrame(fn)
d) queueMicrotask(fn)
e) fetch('/api').then(fn)
f) element.addEventListener('click', fn) — when user clicks
g) new MutationObserver(fn) — when DOM changes
h) requestIdleCallback(fn)
i) new Promise(resolve => resolve()).then(fn)
j) setInterval(fn, 100)
```

<details>
<summary>Answers</summary>

```
a) setTimeout(fn, 0)           → Macrotask queue
b) Promise.resolve().then(fn)  → Microtask queue
c) requestAnimationFrame(fn)   → Animation frame queue (rendering checkpoint)
d) queueMicrotask(fn)          → Microtask queue
e) fetch('/api').then(fn)      → Microtask queue (.then is always microtask)
f) click event handler         → Macrotask queue
g) MutationObserver callback   → Microtask queue
h) requestIdleCallback(fn)     → Idle callback queue
i) new Promise…then(fn)        → Microtask queue (same as b)
j) setInterval(fn, 100)        → Macrotask queue (each interval fires a macrotask)
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/03-event-loop.md`](./03-event-loop.md) — Full event loop mechanics
- [`javascript-core/11-promise-internals.md`](./11-promise-internals.md) — How Promises work internally
- [`rendering/03-cooperative-scheduling.md`](../rendering/03-cooperative-scheduling.md) — Practical scheduling patterns
- [`rendering/05-ui-freezing-solutions.md`](../rendering/05-ui-freezing-solutions.md) — Fixing frozen UIs
- [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md) — The rendering checkpoint in context

---

<div align="center">

**Next:** [`javascript-core/05-closures.md`](./05-closures.md) →

</div>
