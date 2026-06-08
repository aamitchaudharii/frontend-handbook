# 03 — Cooperative Scheduling

> **"Cooperative scheduling is the agreement between your code and the browser: I'll give you back control periodically, and you'll use those moments to render, handle input, and run other tasks. When JavaScript holds the thread without yielding, it breaks that agreement — and users feel it."**

The browser's main thread is single-threaded. JavaScript, layout, paint, event handling — they all compete for the same thread. Cooperative scheduling is the discipline of breaking long tasks into smaller pieces that voluntarily yield control back to the browser between each piece, keeping the thread responsive. This document covers the browser's task model, Long Tasks, yield strategies, the Scheduler API, React's fiber scheduler, and how to design cooperative systems.

---

## 📚 Table of Contents

1. [The Browser's Task Model](#1-the-browsers-task-model)
2. [Long Tasks and INP](#2-long-tasks-and-inp)
3. [What Cooperation Means](#3-what-cooperation-means)
4. [Yield Strategies](#4-yield-strategies)
5. [Time-Sliced Processing](#5-time-sliced-processing)
6. [The Scheduler API](#6-the-scheduler-api)
7. [requestIdleCallback — Background Work](#7-requestidlecallback--background-work)
8. [React Fiber as a Cooperative Scheduler](#8-react-fiber-as-a-cooperative-scheduler)
9. [Cooperative Web Workers](#9-cooperative-web-workers)
10. [Measuring Cooperation](#10-measuring-cooperation)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. The Browser's Task Model

### The Main Thread Task Queue

The browser's main thread processes tasks from a queue one at a time:

```
Main thread task queue:

  [Script eval]   → executes, returns control
  [Event: click]  → handler runs, returns control
  [setTimeout cb] → callback runs, returns control
  [rAF callback]  → runs, returns control
  [Render]        → layout + paint + composite

Between each task:
  Browser checks: do I need to render a frame?
  Browser checks: are there higher-priority tasks (input events)?
  Browser checks: are there microtasks to drain?
```

### The 50ms Long Task Boundary

```
50ms = the boundary the Chrome team established as the maximum
       time a task can run before it noticeably blocks input handling.

Why 50ms?
  Research shows: users perceive delays > 100ms as "laggy"
  Input → response should be < 100ms for "instant" feel
  Browser needs time to: receive input event, handle it, paint
  If a task runs for 50ms, that leaves ~50ms for the browser
  to handle input before the "laggy" threshold is crossed

Tasks > 50ms are "Long Tasks" — logged in Performance panel, measured by PerformanceObserver
Tasks > 200ms are definitively "janky" — user notices
Tasks > 1000ms are "blocking" — user assumes the page is broken
```

### The Rendering Opportunity

```
TASK QUEUE + RENDERING:

  [...task1...] → render → [...task2...] → render → [task3]
                  ↑ 16ms                   ↑ 16ms     ↑ this task will
                  frame budget             frame       miss the render
                                           budget      because it's long

LONG TASK blocks rendering:

  [...task1...] → [...loooooong task 2: 150ms...] → render
                  ↑ scheduled for frame 2           ↑ frame 2 finally happens
                  but frames 2,3,4,5,6,7,8,9 missed  9 dropped frames (150/16 ≈ 9)
```

---

## 2. Long Tasks and INP

### Interaction to Next Paint (INP)

INP measures the time from user interaction (click, key press, tap) to the next visual update. Long tasks directly damage INP:

```
User clicks button:
  t=0ms:   Input event queued
  t=0ms:   Long task is running (50ms+)
  t=50ms:  Long task completes
  t=52ms:  Input event handled
  t=55ms:  Paint
  ──────────────────
  INP: 55ms

For a good INP (< 200ms), the entire path must be fast.
A single long task on the critical path can break INP.
```

### Detecting Long Tasks

```javascript
// PerformanceObserver: detect all long tasks
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      console.warn(`Long task: ${entry.duration.toFixed(1)}ms`);
      // attribution: which script/frame caused it
      const attribution = entry.attribution?.[0];
      if (attribution) {
        console.warn(`  Source: ${attribution.name}`);
        console.warn(`  Container: ${attribution.containerType}`);
      }
    }
  }
});
observer.observe({ entryTypes: ["longtask"] });

// Or use web-vitals library for INP measurement:
import { onINP } from "web-vitals";
onINP(({ value, rating }) => {
  console.log(`INP: ${value}ms (${rating})`);
  // rating: 'good' (<200ms), 'needs-improvement' (<500ms), 'poor' (>500ms)
});
```

---

## 3. What Cooperation Means

### The Contract

```
COOPERATIVE SCHEDULING = each piece of work:
  1. Does its chunk of work
  2. Yields back to the browser/scheduler
  3. Schedules its next chunk
  4. Yields again
  ...repeat...

The browser gets control between chunks to:
  - Handle user input (click, key, scroll)
  - Run pending animations (rAF callbacks)
  - Render frames (layout, paint, composite)
  - Process microtasks

The code makes progress without monopolizing the thread.
```

### Non-Cooperative vs Cooperative

```javascript
// NON-COOPERATIVE: processes all 10,000 items synchronously
// Main thread blocked for entire duration
function processAll(items) {
  for (const item of items) {
    heavyProcess(item); // ~1ms each
  }
  // 10,000 × 1ms = 10,000ms = 10 seconds of blocked main thread!
}

// COOPERATIVE: processes in chunks, yields between each
async function processAllCooperative(items) {
  const CHUNK = 50; // items per chunk
  let processed = 0;

  while (processed < items.length) {
    const end = Math.min(processed + CHUNK, items.length);

    // Process one chunk (~50ms)
    for (let i = processed; i < end; i++) {
      heavyProcess(items[i]);
    }

    processed = end;

    // YIELD: give browser control between chunks
    await yieldToMain();
    // Browser can now: handle input, render, run other tasks
  }
}

// Result:
// Non-cooperative: 10 seconds of jank
// Cooperative: 200 chunks × 50ms = 10 seconds total, but in 50ms slices
//              Browser gets control 199 times during processing
//              Animations stay smooth, input is handled immediately
```

---

## 4. Yield Strategies

### Method 1 — `setTimeout(fn, 0)`

```javascript
function yieldToMain() {
  return new Promise((resolve) => setTimeout(resolve, 0));
}

// How it works:
// setTimeout(fn, 0) schedules fn as a macrotask
// The current task completes → browser gets control → fn runs as next task
// Any input events, renders, etc. can happen between them

// Usage:
async function processChunked(items) {
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    processChunk(items.slice(i, i + CHUNK_SIZE));
    await yieldToMain();
  }
}

// Downside: minimum ~4ms delay per yield (setTimeout minimum)
// For latency-sensitive work: use scheduler.yield() if available
```

### Method 2 — `MessageChannel`

```javascript
// MessageChannel: faster than setTimeout (no 4ms minimum delay)
function yieldToMain() {
  return new Promise((resolve) => {
    const { port1, port2 } = new MessageChannel();
    port1.onmessage = resolve;
    port2.postMessage(null);
    // Runs as a task, like setTimeout(fn, 0) but without the timer clamping
  });
}
```

### Method 3 — `scheduler.yield()` (Modern, Chrome 115+)

```javascript
// scheduler.yield(): designed exactly for this purpose
// Yields control to the browser, resumes with the same task priority

async function processChunked(items) {
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    processChunk(items.slice(i, i + CHUNK_SIZE));

    if ("scheduler" in window && "yield" in scheduler) {
      await scheduler.yield(); // native, most efficient
    } else {
      await new Promise((r) => setTimeout(r, 0)); // fallback
    }
  }
}
```

### Method 4 — `requestIdleCallback` for Background Work

```javascript
// rIC: yield AND only resume during idle time
// Use for non-critical background work
function processInIdle(items) {
  let index = 0;

  function processMore(deadline) {
    // Process while there's idle time remaining
    while (index < items.length && deadline.timeRemaining() > 1) {
      heavyProcess(items[index++]);
    }

    if (index < items.length) {
      requestIdleCallback(processMore, { timeout: 1000 });
    }
  }

  requestIdleCallback(processMore);
}
```

### Choosing a Yield Strategy

```
User-interactive work (clicks, animations):
  → scheduler.yield() or MessageChannel
  → Yield frequently (every 5-10ms)
  → Prioritize responsiveness over throughput

Background processing (parse, sort, index):
  → requestIdleCallback
  → Yield when deadline.timeRemaining() < 1ms
  → Prioritize not competing with UI

Balanced processing (data transformation, rendering):
  → setTimeout(fn, 0) or scheduler.yield() every 50ms
  → Balance throughput and responsiveness
```

---

## 5. Time-Sliced Processing

Time slicing processes work in time-bounded chunks rather than count-bounded chunks:

```javascript
// COUNT-BASED CHUNKING:
// Process N items per chunk
// Problem: heavy items blow the budget, light items underutilize it

// TIME-BASED CHUNKING:
// Process for T milliseconds per chunk, then yield
// Items per chunk varies based on their cost

/**
 * Process items in time-sliced chunks.
 * @param items Array of items to process
 * @param processFn Function to call for each item
 * @param budgetMs Target time budget per chunk (ms)
 * @param signal AbortSignal for cancellation
 */
async function processTimeSliced<T, R>(
  items:     T[],
  processFn: (item: T) => R,
  options: {
    budgetMs?:  number;
    onProgress?: (done: number, total: number) => void;
    signal?:    AbortSignal;
  } = {}
): Promise<R[]> {
  const { budgetMs = 8, onProgress, signal } = options;
  const results: R[] = new Array(items.length);
  let index = 0;

  while (index < items.length) {
    if (signal?.aborted) throw new DOMException('Aborted', 'AbortError');

    const chunkStart = performance.now();

    // Process until budget is exceeded
    while (index < items.length && performance.now() - chunkStart < budgetMs) {
      results[index] = processFn(items[index]);
      index++;
    }

    onProgress?.(index, items.length);

    // Yield if there's more work
    if (index < items.length) {
      await yieldToMain();
    }
  }

  return results;
}

// Usage: process 100,000 items without blocking
const results = await processTimeSliced(
  largeDataset,
  (item) => expensiveTransform(item),
  {
    budgetMs:   8,  // use up to 8ms per chunk (half of 16ms frame)
    onProgress: (done, total) => {
      progressBar.style.width = `${(done / total * 100).toFixed(0)}%`;
    },
  }
);
```

### Adaptive Chunk Sizing

```javascript
// Adaptive: adjust chunk size based on observed processing speed
async function processAdaptive(items, processFn, targetMs = 8) {
  const results = [];
  let chunkSize = 100; // start with 100 items
  let index = 0;

  while (index < items.length) {
    const start = performance.now();
    const end = Math.min(index + chunkSize, items.length);

    for (let i = index; i < end; i++) {
      results[i] = processFn(items[i]);
    }

    const elapsed = performance.now() - start;
    index = end;

    // Tune chunk size: if chunk was fast, increase; if slow, decrease
    if (elapsed > 0) {
      const itemsPerMs = (end - index + chunkSize) / elapsed;
      chunkSize = Math.max(1, Math.floor(itemsPerMs * targetMs));
    }

    if (index < items.length) {
      await yieldToMain();
    }
  }

  return results;
}
```

---

## 6. The Scheduler API

The Scheduler API (`scheduler.postTask()`) provides explicit task priority management:

```javascript
// Task priorities (Chrome's implementation)
// 'user-blocking': highest — for work that affects responsiveness
// 'user-visible':  default — for work the user will see
// 'background':    lowest — for prefetch, analytics, cleanup

// Schedule a high-priority task
await scheduler.postTask(
  () => {
    updateUserInterface(); // must be fast to not block input
  },
  { priority: "user-blocking" },
);

// Schedule a normal priority task
await scheduler.postTask(
  () => {
    renderSecondaryContent();
  },
  { priority: "user-visible" },
);

// Schedule a background task
scheduler.postTask(
  () => {
    prefetchNextPage();
    updateAnalytics();
  },
  { priority: "background" },
);
```

### Task Queues in Practice

```javascript
class ScheduledTaskQueue {
  #tasks = {
    "user-blocking": [],
    "user-visible": [],
    background: [],
  };
  #running = false;

  enqueue(task, priority = "user-visible") {
    this.#tasks[priority].push(task);
    if (!this.#running) this.#drain();
  }

  async #drain() {
    this.#running = true;

    while (this.#hasTasks()) {
      // Always process highest-priority tasks first
      const priority = this.#getHighestPriorityWithWork();
      const task = this.#tasks[priority].shift();

      if (task) {
        await scheduler.postTask(task, { priority });
      }
    }

    this.#running = false;
  }

  #hasTasks() {
    return Object.values(this.#tasks).some((q) => q.length > 0);
  }

  #getHighestPriorityWithWork() {
    if (this.#tasks["user-blocking"].length) return "user-blocking";
    if (this.#tasks["user-visible"].length) return "user-visible";
    return "background";
  }
}

const taskQueue = new ScheduledTaskQueue();

// Enqueue tasks at appropriate priorities
taskQueue.enqueue(() => handleUserClick(), "user-blocking");
taskQueue.enqueue(() => renderFeed(), "user-visible");
taskQueue.enqueue(() => prefetchImages(), "background");
```

---

## 7. requestIdleCallback — Background Work

`requestIdleCallback` (rIC) schedules work during browser idle time — after all high-priority tasks and rendering are done.

```javascript
// rIC fires during idle time:
// - After all input events handled
// - After animation frame callbacks
// - After layout and paint
// - When browser has "nothing better to do"

requestIdleCallback(
  (deadline) => {
    console.log("Idle time available:", deadline.timeRemaining(), "ms");
    console.log("Was forced by timeout:", deadline.didTimeout);

    // deadline.timeRemaining(): ms left in current idle period
    // Can be 0ms (very busy) to ~50ms (quiet periods)
    // deadline.didTimeout: true if forced by the timeout option

    // Process as much as possible within the time available
    while (workQueue.length > 0 && deadline.timeRemaining() > 1) {
      const task = workQueue.shift();
      task();
    }

    // Schedule more if work remains
    if (workQueue.length > 0) {
      requestIdleCallback(this, { timeout: 2000 });
    }
  },
  {
    timeout: 2000, // force-run within 2 seconds even if not idle
  },
);
```

### rIC vs rAF vs setTimeout

```
requestIdleCallback:
  When: during idle time (after rendering)
  Use for: non-critical background work
  Don't use for: any user-visible updates, anything time-sensitive

requestAnimationFrame:
  When: before next paint (within frame budget)
  Use for: visual updates, animations
  Don't use for: heavy computation, non-visual work

setTimeout(fn, 0):
  When: as next macrotask (may interrupt rendering)
  Use for: breaking up synchronous work cooperatively
  Minimum delay: ~4ms (timer clamping)
```

---

## 8. React Fiber as a Cooperative Scheduler

React Fiber implements its own cooperative scheduler to make rendering interruptible.

### How Fiber Yields

```javascript
// React's scheduler.js (simplified)
const FRAME_YIELD_MS = 5; // yield after 5ms of work

let deadline = 0;

function shouldYield() {
  return performance.now() >= deadline;
}

function workLoop(hasTimeRemaining) {
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
    // Each "unit of work" is one fiber (component)
  }
}

// Before processing starts:
function scheduleCallback(priority, callback) {
  deadline = performance.now() + FRAME_YIELD_MS;
  // Process fibers until 5ms elapsed, then yield
}

// React yields using MessageChannel (faster than setTimeout):
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  // Resumed after yielding
  const currentTime = performance.now();
  if (currentTime < deadline) {
    // More time available: continue processing
    workLoop(true);
  } else {
    // No time left: yield again
    scheduleMoreWork();
  }
};
```

### React's Priority Lanes and Scheduling

```
React's scheduler priority levels:
  ImmediatePriority:  synchronous, must run now (legacy mode)
  UserBlockingPriority: 250ms timeout (user input)
  NormalPriority:     5s timeout (data loading, transitions)
  LowPriority:        10s timeout (background work)
  IdlePriority:       never expires (prefetching)

When startTransition is used:
  - The update is tagged as TransitionLane (NormalPriority)
  - If higher-priority work comes in (user types again):
    React stops processing the transition
    Processes the user input (UserBlockingPriority)
    Returns to the transition
  - User always gets responsive input, even during expensive renders
```

### Fiber Work Loop in Detail

```
RENDER PHASE (interruptible):
  workInProgress = current root
  while (workInProgress && !shouldYield()):
    beginWork(fiber)    → call component function, get new children
    completeWork(fiber) → create DOM nodes if needed
    workInProgress = next fiber

COMMIT PHASE (synchronous, not interruptible):
  commitMutationEffects()  → apply DOM mutations
  commitLayoutEffects()    → call useLayoutEffect
  commitPassiveEffects()   → call useEffect (async)
```

---

## 9. Cooperative Web Workers

Workers offload heavy computation entirely, eliminating the need to yield on the main thread.

```javascript
// Main thread: fire and forget, stays cooperative
const worker = new Worker("./heavy-worker.js", { type: "module" });

async function processOffThread(data) {
  return new Promise((resolve, reject) => {
    const messageId = generateId();

    worker.onmessage = ({ data: result }) => {
      if (result.id === messageId) {
        resolve(result.output);
      }
    };

    // Transfer ArrayBuffer ownership (zero-copy)
    const buffer = serializeToBuffer(data);
    worker.postMessage({ id: messageId, buffer }, [buffer]);
  });
}

// Main thread never blocks — worker does all the work
const result = await processOffThread(largeDataset);
updateUI(result); // only this runs on main thread
```

```javascript
// Worker: can use blocking algorithms since it doesn't affect main thread
// heavy-worker.js
self.onmessage = ({ data }) => {
  const input = deserializeFromBuffer(data.buffer);

  // No need to yield — workers don't block main thread rendering
  const result = expensiveAlgorithm(input); // 500ms — fine in worker!

  const outputBuffer = serializeToBuffer(result);
  self.postMessage({ id: data.id, buffer: outputBuffer }, [outputBuffer]);
};
```

### When Workers Beat Chunking

```
CHUNKING (main thread, yielding):
  Pro: simpler, can access DOM
  Con: still competes with rendering/input for thread time
  Con: 5-50ms per chunk × 200 chunks = many small interruptions
  Best for: work that needs DOM access, quick operations

WORKERS (separate thread):
  Pro: zero impact on main thread
  Pro: can use blocking algorithms (simpler code)
  Pro: parallelism (multiple workers)
  Con: no DOM access
  Con: serialization cost (postMessage)
  Best for: CPU-heavy computation, sorting, filtering, ML inference
```

---

## 10. Measuring Cooperation

### Long Task Observer

```javascript
// Detect when tasks exceed 50ms
const longTaskObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      console.warn(
        `Long task detected: ${entry.duration.toFixed(1)}ms`,
        "\nStart:",
        new Date(performance.timeOrigin + entry.startTime).toISOString(),
        "\nAttribution:",
        entry.attribution?.[0],
      );
    }
  }
});
longTaskObserver.observe({ entryTypes: ["longtask"] });
```

### INP Monitoring

```javascript
import { onINP } from "web-vitals/attribution";

onINP(({ value, rating, attribution }) => {
  console.log(`INP: ${value}ms (${rating})`);

  if (attribution) {
    console.log("Interaction type:", attribution.interactionType);
    console.log("Interaction target:", attribution.interactionTarget);
    console.log("Event time:", attribution.interactionTime);
    console.log("Next paint:", attribution.nextPaintTime);
    console.log("Event duration:", attribution.eventDuration);
    console.log("Processing duration:", attribution.processingDuration);
    console.log("Presentation delay:", attribution.presentationDelay);
  }

  // Report to analytics
  analytics.track("inp", {
    value,
    rating,
    interaction_type: attribution?.interactionType,
  });
});
```

### Custom Performance Budget Enforcement

```javascript
class CooperativeEnforcer {
  #maxTaskMs;
  #taskStart = null;

  constructor(maxTaskMs = 50) {
    this.#maxTaskMs = maxTaskMs;
    this.#monitor();
  }

  #monitor() {
    // Patch setTimeout to track when tasks start
    const origSetTimeout = window.setTimeout;
    window.setTimeout = (...args) => {
      const wrappedCallback = () => {
        this.#taskStart = performance.now();
        args[0]();
        const duration = performance.now() - this.#taskStart;
        if (duration > this.#maxTaskMs) {
          console.warn(`Task exceeded budget: ${duration.toFixed(1)}ms`);
        }
      };
      return origSetTimeout(wrappedCallback, args[1], ...args.slice(2));
    };
  }
}
```

---

## 11. Good Practices

### ✅ Measure first — don't yield speculatively

```javascript
// ✅ Only add yielding when you've measured a Long Task problem
// Premature yielding adds complexity and overhead with no benefit

// Profile → find long tasks → add targeted yields
// DON'T add yields to every loop "just in case"
```

### ✅ Use `scheduler.yield()` for user-interaction-critical paths

```javascript
// ✅ Yield after each chunk to allow input handling
async function renderFeed(posts) {
  for (const post of posts) {
    renderPost(post); // ~5ms per post

    if ("scheduler" in window && "yield" in scheduler) {
      await scheduler.yield();
    }
    // User can click/scroll between each post render
  }
}
```

### ✅ Use workers for CPU-heavy computation

```javascript
// ✅ Offload to worker — main thread stays free entirely
const sortWorker = new Worker("./sort-worker.js");
const sorted = await sortInWorker(sortWorker, largeDataset);
// No yielding needed — never touched the main thread
```

### ✅ Set meaningful timeouts on `requestIdleCallback`

```javascript
// ✅ Timeout ensures work eventually runs even in busy scenarios
requestIdleCallback(doBackgroundWork, {
  timeout: 3000, // run within 3 seconds even if never truly idle
});
// Without timeout: work may never run on very busy pages
```

---

## 12. Bad Practices

### ❌ Large synchronous data processing on the main thread

```javascript
// ❌ 100,000 item sort on main thread — can take 500ms+
const sorted = largeArray.sort(complexComparator);
// Entire main thread blocked for 500ms — severe jank

// ✅ Worker for heavy computation
const sorted = await sortInWorker(largeArray);
```

### ❌ `requestIdleCallback` for user-visible work

```javascript
// ❌ rIC can be delayed indefinitely
requestIdleCallback(() => {
  updateSearchResults(); // user is waiting for this!
});
// If the browser is busy: callback may not fire for seconds

// ✅ User-visible work should use rAF or scheduler.postTask('user-visible')
requestAnimationFrame(() => updateSearchResults());
```

### ❌ Yielding too frequently (micro-yield)

```javascript
// ❌ Yielding every single item — overhead exceeds benefit
async function processItems(items) {
  for (const item of items) {
    process(item); // 0.1ms per item
    await yieldToMain(); // 4ms delay — 40× slower than the work!
  }
}

// ✅ Yield in time-based chunks
async function processItems(items) {
  let i = 0;
  while (i < items.length) {
    const start = performance.now();
    while (i < items.length && performance.now() - start < 8) {
      process(items[i++]);
    }
    await yieldToMain(); // yield after 8ms of work
  }
}
```

---

## 13. Common Mistakes

### Mistake 1 — Mixing async/await with synchronous blocking code

```javascript
// ❌ await doesn't prevent blocking — it only yields between awaits
async function badCooperation(items) {
  for (let i = 0; i < items.length; i++) {
    await processChunk(items, i); // 100ms of synchronous work inside!
    // The await comes AFTER the 100ms block — too late
  }
}

// ✅ await before the work, not after
async function goodCooperation(items) {
  for (let i = 0; i < items.length; i++) {
    await yieldToMain(); // yield BEFORE each chunk
    processChunk(items, i); // now runs as its own task
  }
}
```

### Mistake 2 — Cancellation not implemented

```javascript
// ❌ Long cooperative task with no cancellation
async function processAll(items) {
  for (const chunk of chunks(items, 50)) {
    processChunk(chunk);
    await yieldToMain();
    // What if user navigates away? Task continues indefinitely!
  }
}

// ✅ AbortController for cancellation
async function processAll(items, signal) {
  for (const chunk of chunks(items, 50)) {
    if (signal.aborted) throw new DOMException("Aborted", "AbortError");
    processChunk(chunk);
    await yieldToMain();
  }
}

// Usage:
const controller = new AbortController();
processAll(items, controller.signal);

// Cancel when component unmounts:
return () => controller.abort();
```

### Mistake 3 — Priority inversion

```javascript
// ❌ Low-priority work scheduled before high-priority work
scheduler.postTask(updateAnalytics, { priority: "background" }); // goes first!
scheduler.postTask(renderUserInput, { priority: "user-blocking" }); // goes second

// ✅ High-priority tasks run first regardless of scheduling order
// scheduler.postTask executes by priority, not scheduling order
// This is correct behavior — but make sure critical work uses the right priority
```

---

## 14. Interview-Level Explanation

> **"What is cooperative scheduling? How does it relate to web performance?"**

**Strong answer:**

> "Cooperative scheduling is the practice of breaking long JavaScript tasks into smaller pieces that voluntarily yield control back to the browser between each piece. The browser's main thread is single-threaded — JavaScript, layout, paint, and input event handling all share it. When JavaScript holds the thread for more than 50ms without yielding, it's called a Long Task. During a Long Task, the browser can't handle user input or render new frames — users experience this as frozen UI, unresponsive clicks, and jank.
>
> The key metric is INP — Interaction to Next Paint. If a user clicks during a 200ms Long Task, their input can't be processed until the task finishes. Their click response might take 220ms, well above the 200ms 'good' threshold. Cooperative scheduling prevents this by breaking the Long Task into 8-16ms chunks with yields between them.
>
> There are several yield mechanisms. `scheduler.yield()` is the most appropriate — it was designed exactly for this purpose and preserves task priority. `setTimeout(fn, 0)` works as a fallback but has a minimum 4ms delay. `requestIdleCallback` is for background work that can wait for idle time — not for user-visible updates.
>
> The cooperative pattern looks like: check if budget is exceeded, if so `await yieldToMain()`, then continue. The yield is asynchronous — it returns control to the browser as a macrotask, allowing input events and rendering to happen before resuming.
>
> For truly heavy computation, the better solution is Web Workers. Workers run on a separate thread entirely — no yielding needed because the main thread is never touched. The tradeoff is no DOM access and serialization cost for postMessage, but for things like sorting 100,000 records or building a search index, workers are clearly superior.
>
> React's Fiber architecture implements its own cooperative scheduler — it processes one component at a time and checks after each whether to yield, enabling high-priority work (user input) to interrupt low-priority renders (transitions)."

---

## 15. Exercises

### Exercise 1 — Time-slice a search index build

You need to build a search index for 50,000 documents. The index build takes about 2ms per document (100 seconds total). Implement a cooperative version that:

- Shows progress to the user
- Can be cancelled (e.g., when user navigates away)
- Uses 8ms time slices to keep the main thread responsive

```typescript
interface Document {
  id: string;
  title: string;
  content: string;
}
interface SearchIndex {
  add(doc: Document): void;
  search(query: string): string[];
}

async function buildIndex(
  documents: Document[],
  index: SearchIndex,
  onProgress: (pct: number) => void,
  signal: AbortSignal,
): Promise<void> {
  // Your implementation here
}
```

<details>
<summary>Solution</summary>

```typescript
async function buildIndex(
  documents: Document[],
  index: SearchIndex,
  onProgress: (pct: number) => void,
  signal: AbortSignal,
): Promise<void> {
  let i = 0;

  while (i < documents.length) {
    if (signal.aborted) {
      throw new DOMException("Index build cancelled", "AbortError");
    }

    const chunkStart = performance.now();

    // Process until 8ms budget exceeded
    while (i < documents.length && performance.now() - chunkStart < 8) {
      index.add(documents[i]);
      i++;
    }

    // Report progress after each chunk
    onProgress((i / documents.length) * 100);

    // Yield if more work remains
    if (i < documents.length) {
      // Use scheduler.yield() if available, fallback to setTimeout
      if ("scheduler" in window && "yield" in (window as any).scheduler) {
        await (window as any).scheduler.yield();
      } else {
        await new Promise((resolve) => setTimeout(resolve, 0));
      }
    }
  }

  onProgress(100);
}

// Usage:
const controller = new AbortController();

await buildIndex(
  allDocuments,
  searchIndex,
  (pct) => (progressBar.style.width = `${pct}%`),
  controller.signal,
);

// To cancel:
controller.abort();
```

</details>

---

## 🔗 Related Topics

- [`rendering/01-dom-batching.md`](./01-dom-batching.md) — Batching DOM operations
- [`rendering/02-virtual-dom.md`](./02-virtual-dom.md) — React Fiber's cooperative rendering
- [`performance/04-raf-optimization.md`](../performance/04-raf-optimization.md) — rAF in the scheduling pipeline
- [`javascript-core/12-web-workers.md`](../javascript-core/12-web-workers.md) — Workers for off-thread computation
- [`performance/12-large-data-rendering.md`](../performance/12-large-data-rendering.md) — Chunked processing in practice

---

<div align="center">

**Next:** [`rendering/04-paint-optimization.md`](./04-paint-optimization.md) →

</div>
