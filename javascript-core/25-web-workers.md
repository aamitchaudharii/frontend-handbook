# 12 — Web Workers

> **"JavaScript is single-threaded. Web Workers are how you escape that constraint — real OS threads running real JavaScript, in parallel, without blocking the UI."**

Web Workers give JavaScript true parallelism. Not the cooperative concurrency of async/await — actual simultaneous execution on separate CPU cores. This document covers dedicated workers, shared workers, the message passing model, transferable objects, worker pools, and every pattern you need to offload heavy work without freezing the UI.

---

## 📚 Table of Contents

1. [Why Web Workers Exist](#1-why-web-workers-exist)
2. [The Three Worker Types](#2-the-three-worker-types)
3. [Dedicated Workers — Deep Dive](#3-dedicated-workers--deep-dive)
4. [The Message Passing Model](#4-the-message-passing-model)
5. [Transferable Objects — Zero-Copy Transfer](#5-transferable-objects--zero-copy-transfer)
6. [SharedArrayBuffer and Atomics](#6-sharedarraybuffer-and-atomics)
7. [Worker Lifecycle and Memory](#7-worker-lifecycle-and-memory)
8. [Worker Pools — Production Pattern](#8-worker-pools--production-pattern)
9. [Shared Workers](#9-shared-workers)
10. [Service Workers](#10-service-workers-overview)
11. [What Workers Can and Cannot Do](#11-what-workers-can-and-cannot-do)
12. [Comlink — RPC Over Workers](#12-comlink--rpc-over-workers)
13. [Real-World Use Cases](#13-real-world-use-cases)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. Why Web Workers Exist

The main thread in a browser is responsible for everything: JavaScript execution, style calculation, layout, paint, compositing, and handling user input. When JavaScript runs a long synchronous task on the main thread, all of these are blocked.

```
MAIN THREAD — without workers:

Time:  0ms    100ms   200ms   300ms   400ms   500ms
       │      │       │       │       │       │
JS:    ████████████████████████████████████████  ← 500ms heavy computation
UI:    ████████████████████████████████████████  ← FROZEN for 500ms
Input: ████████████████████████████████████████  ← clicks ignored

MAIN THREAD — with worker:

Main:  │ send data │           │ receive result │
       ├───────────┤───────────┤────────────────┤
       │           │ UI free!  │                │ ← renders, handles input
       │           │           │                │

Worker:│           │ computing │                │ ← 500ms computation on its own thread
```

The key insight: workers run on **separate OS threads** — not just different tasks in the event loop. The OS scheduler can execute them on different CPU cores simultaneously.

---

## 2. The Three Worker Types

```
┌──────────────────────────────────────────────────────────────────┐
│  DEDICATED WORKER                                                 │
│  - One-to-one: one page ↔ one worker                            │
│  - Simplest model                                                │
│  - Created with: new Worker('./worker.js')                       │
│  - Terminated when page closes or worker.terminate() called     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  SHARED WORKER                                                    │
│  - One-to-many: multiple tabs/windows ↔ one worker              │
│  - Shared state across tabs from the same origin                │
│  - Created with: new SharedWorker('./worker.js')                 │
│  - Stays alive as long as any connected page is open            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  SERVICE WORKER                                                   │
│  - Acts as a network proxy between page and server               │
│  - Enables offline support, push notifications, background sync  │
│  - Created via: navigator.serviceWorker.register('./sw.js')      │
│  - Separate lifecycle — persists beyond page close               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Dedicated Workers — Deep Dive

### Creating a Worker

```javascript
// main.js
const worker = new Worker("./heavy-worker.js");
// or with a module worker (ES module syntax in worker):
const worker = new Worker("./heavy-worker.js", { type: "module" });
```

### Inline Worker (no separate file)

```javascript
// Create worker from a string — useful for self-contained workers
const workerCode = `
  self.onmessage = function(e) {
    const result = heavyComputation(e.data);
    self.postMessage(result);
  };

  function heavyComputation(n) {
    let sum = 0;
    for (let i = 0; i < n; i++) sum += Math.sqrt(i);
    return sum;
  }
`;

const blob = new Blob([workerCode], { type: "application/javascript" });
const url = URL.createObjectURL(blob);
const worker = new Worker(url);

// Cleanup the object URL when done
worker.addEventListener("message", () => URL.revokeObjectURL(url));
```

### Basic Communication

```javascript
// main.js
const worker = new Worker("./worker.js");

// Send message to worker
worker.postMessage({ type: "compute", data: [1, 2, 3, 4, 5] });

// Receive message from worker
worker.onmessage = (event) => {
  console.log("Result:", event.data);
};

// Handle worker errors
worker.onerror = (error) => {
  console.error(
    "Worker error:",
    error.message,
    "at",
    error.filename,
    ":",
    error.lineno,
  );
};
```

```javascript
// worker.js
// Inside a worker, `self` refers to the worker's global scope (DedicatedWorkerGlobalScope)

self.onmessage = (event) => {
  const { type, data } = event.data;

  if (type === "compute") {
    const result = data.reduce((sum, n) => sum + Math.sqrt(n), 0);
    self.postMessage({ type: "result", result });
  }
};

// Workers can also import scripts (classic workers):
importScripts("./utils.js", "./math.js");

// Or use ES module imports (module workers):
// import { compute } from './utils.js'; // with type: 'module'
```

---

## 4. The Message Passing Model

Workers and the main thread communicate exclusively through messages. There is **no shared memory by default** — all data is copied (structured clone) when passed between threads.

### Structured Clone — The Default

```javascript
// postMessage uses the Structured Clone Algorithm
// It can handle:
const data = {
  primitives: { n: 42, s: "hello", b: true, nil: null },
  objects: { nested: { deep: true } },
  arrays: [1, 2, 3],
  dates: new Date(),
  maps: new Map([["key", "value"]]),
  sets: new Set([1, 2, 3]),
  typedArrays: new Float32Array([1.0, 2.0, 3.0]),
  arrayBuffers: new ArrayBuffer(1024),
  errors: new Error("test"),
  // RegExp, Blob, File, ImageData, etc.
};

worker.postMessage(data); // deep copied — worker gets an independent copy
```

### What Structured Clone CANNOT Copy

```javascript
// These will throw or be silently dropped:
worker.postMessage(() => {}); // DataCloneError: functions not cloneable
worker.postMessage(document.getElementById("div")); // DOM nodes not cloneable
worker.postMessage(window); // Window not cloneable
class MyClass {}
worker.postMessage(new MyClass()); // prototype methods stripped
```

### Message Cost — Clone Overhead

Structured clone is not free. For large data structures, the copy takes measurable time:

```javascript
// Benchmark: cloning cost for large arrays
const largeArray = new Array(1_000_000)
  .fill(0)
  .map((_, i) => ({ id: i, value: Math.random() }));

console.time("postMessage clone");
worker.postMessage(largeArray); // copies 1M objects across thread boundary
console.timeEnd("postMessage clone");
// Typical: 50-200ms for 1M complex objects — expensive!
```

This is why **transferable objects** exist — for zero-copy transfer.

### Message Protocol Design

For any non-trivial worker, use a typed message protocol:

```javascript
// Define message types as constants
const MSG = {
  // Main → Worker
  COMPUTE: "COMPUTE",
  CANCEL: "CANCEL",
  CONFIG: "CONFIG",
  // Worker → Main
  RESULT: "RESULT",
  PROGRESS: "PROGRESS",
  ERROR: "ERROR",
  READY: "READY",
};

// main.js
worker.postMessage({ type: MSG.COMPUTE, id: "job-1", payload: data });

// worker.js
self.onmessage = ({ data: { type, id, payload } }) => {
  switch (type) {
    case MSG.COMPUTE:
      try {
        const result = heavyWork(payload);
        self.postMessage({ type: MSG.RESULT, id, result });
      } catch (err) {
        self.postMessage({ type: MSG.ERROR, id, error: err.message });
      }
      break;
    case MSG.CANCEL:
      cancelWork(id);
      break;
  }
};
```

### Request-Response Pattern with Promises

```javascript
// Wrap worker messages in Promises for cleaner async code
class WorkerClient {
  #worker;
  #pending = new Map(); // id → { resolve, reject }
  #nextId = 0;

  constructor(scriptUrl) {
    this.#worker = new Worker(scriptUrl);
    this.#worker.onmessage = ({ data }) => this.#handleMessage(data);
    this.#worker.onerror = (err) => this.#handleError(err);
  }

  send(type, payload) {
    return new Promise((resolve, reject) => {
      const id = `msg-${this.#nextId++}`;
      this.#pending.set(id, { resolve, reject });
      this.#worker.postMessage({ type, id, payload });
    });
  }

  #handleMessage({ type, id, result, error }) {
    const pending = this.#pending.get(id);
    if (!pending) return;

    this.#pending.delete(id);

    if (type === "ERROR" || error) {
      pending.reject(new Error(error));
    } else {
      pending.resolve(result);
    }
  }

  #handleError(err) {
    // Reject all pending with the worker error
    for (const [id, { reject }] of this.#pending) {
      reject(new Error(err.message));
      this.#pending.delete(id);
    }
  }

  terminate() {
    this.#worker.terminate();
    for (const { reject } of this.#pending.values()) {
      reject(new Error("Worker terminated"));
    }
    this.#pending.clear();
  }
}

// Usage
const client = new WorkerClient("./heavy-worker.js");
const result = await client.send("COMPUTE", largeDataset);
```

---

## 5. Transferable Objects — Zero-Copy Transfer

Instead of copying data, **transferable objects** are moved from one thread to another. The original reference becomes neutered (unusable) — ownership transfers.

### Why Zero-Copy Matters

```javascript
// Sending 10MB ArrayBuffer
const buffer = new ArrayBuffer(10 * 1024 * 1024); // 10MB

// With clone (default):
worker.postMessage(buffer);
// Time: ~10ms (copies 10MB across thread boundary)
// Memory: 20MB temporarily (original + copy exist simultaneously)

// With transfer:
worker.postMessage(buffer, [buffer]);
// Time: ~0.1ms (just a pointer swap in memory)
// Memory: 10MB (ownership moved — buffer now belongs to worker)
console.log(buffer.byteLength); // 0 — buffer is neutered (transferred away)
```

### Transferable Types

```javascript
// Types that support transfer (zero-copy):
const ab = new ArrayBuffer(1024);
const f32 = new Float32Array(ab); // shares the underlying ArrayBuffer
const canvas = new OffscreenCanvas(800, 600);
const port = new MessageChannel().port1;
const stream = new ReadableStream(/* ... */);
const bitmap = await createImageBitmap(imgElement);

// Transfer syntax: postMessage(data, [transferList])
worker.postMessage(ab, [ab]); // transfer ArrayBuffer
worker.postMessage(bitmap, [bitmap]); // transfer ImageBitmap
worker.postMessage({ port }, [port]); // transfer MessagePort
```

### Practical Example — Image Processing

```javascript
// main.js: send raw image pixels to worker for processing
async function processImage(imageElement) {
  const canvas = document.createElement("canvas");
  canvas.width = imageElement.naturalWidth;
  canvas.height = imageElement.naturalHeight;
  const ctx = canvas.getContext("2d");
  ctx.drawImage(imageElement, 0, 0);

  // Get pixel data
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  const buffer = imageData.data.buffer; // ArrayBuffer of pixel data

  // Transfer (not copy) the pixel buffer to the worker
  worker.postMessage(
    {
      type: "PROCESS_IMAGE",
      buffer,
      width: imageData.width,
      height: imageData.height,
    },
    [buffer],
  ); // ← transfer list

  // buffer is now neutered — don't use it here
}
```

```javascript
// worker.js: receives and processes the image pixels
self.onmessage = ({ data: { type, buffer, width, height } }) => {
  if (type !== "PROCESS_IMAGE") return;

  const pixels = new Uint8ClampedArray(buffer);

  // Apply grayscale filter — modifies pixels in-place
  for (let i = 0; i < pixels.length; i += 4) {
    const gray =
      0.299 * pixels[i] + 0.587 * pixels[i + 1] + 0.114 * pixels[i + 2];
    pixels[i] = pixels[i + 1] = pixels[i + 2] = gray;
  }

  // Transfer the processed buffer back
  self.postMessage(
    {
      type: "IMAGE_PROCESSED",
      buffer: pixels.buffer,
      width,
      height,
    },
    [pixels.buffer],
  ); // ← transfer back
};
```

---

## 6. SharedArrayBuffer and Atomics

For cases where both threads need to read and write the same memory simultaneously, `SharedArrayBuffer` allows true shared memory between the main thread and workers.

### Creating Shared Memory

```javascript
// Main thread: create shared buffer
const sharedBuffer = new SharedArrayBuffer(4 * 4); // 4 floats × 4 bytes
const sharedArray = new Float32Array(sharedBuffer);

// Initialize
sharedArray[0] = 1.0;
sharedArray[1] = 2.0;

// Send to worker (copied by reference — same underlying memory!)
worker.postMessage({ type: "INIT", buffer: sharedBuffer });
// Note: SharedArrayBuffer is sent by REFERENCE, not copied
// Both threads now access THE SAME MEMORY
```

```javascript
// Worker: receives and accesses the same memory
self.onmessage = ({ data: { type, buffer } }) => {
  if (type !== "INIT") return;
  const shared = new Float32Array(buffer);

  // Reading and writing the SAME memory as main thread
  shared[0] += 10; // visible to main thread immediately
};
```

### Race Conditions — The Problem

Without synchronization, concurrent reads and writes to shared memory produce undefined behavior:

```javascript
// ❌ Race condition — both threads modify shared[0] simultaneously
// Thread A:  read shared[0] (= 5)
// Thread B:  read shared[0] (= 5)
// Thread A:  write shared[0] = 5 + 1 = 6
// Thread B:  write shared[0] = 5 + 1 = 6  ← lost update! Should be 7
```

### Atomics — Safe Concurrent Operations

`Atomics` provides **atomic operations** — guaranteed indivisible reads/writes that prevent race conditions:

```javascript
// Atomic operations on SharedArrayBuffer
const sab = new SharedArrayBuffer(4);
const int32 = new Int32Array(sab);

// Atomic read (no partial reads)
const value = Atomics.load(int32, 0);

// Atomic write (no partial writes)
Atomics.store(int32, 0, 42);

// Atomic add — returns OLD value
const oldValue = Atomics.add(int32, 0, 1); // old = 0, new = 1

// Atomic compare-and-swap (CAS) — the foundation of lock-free algorithms
// If int32[0] === expectedValue, set it to newValue and return expectedValue
// Otherwise, do nothing and return current value
const expected = 1;
const result = Atomics.compareExchange(int32, 0, expected, 99);
// result = old value; int32[0] = 99 (if was 1) or unchanged (if wasn't 1)

// Atomics.wait / Atomics.notify — mutex-like blocking (workers only, not main thread)
// Worker can wait for a value to change:
Atomics.wait(int32, 0, 0); // block until int32[0] !== 0
// Main thread notifies:
Atomics.notify(int32, 0, 1); // wake up 1 waiting thread
```

### Practical: Progress Reporting via SharedArrayBuffer

```javascript
// Use SharedArrayBuffer for high-frequency progress updates
// (avoids postMessage overhead for every progress tick)

// main.js
const progressBuffer = new SharedArrayBuffer(8); // 2 × Int32
const progress = new Int32Array(progressBuffer);
// progress[0] = items processed
// progress[1] = total items

worker.postMessage({ type: "START", progressBuffer, data: largeDataset });

// Poll progress at 60fps without postMessage overhead
function watchProgress() {
  const done = Atomics.load(progress, 0);
  const total = Atomics.load(progress, 1);
  updateProgressBar(done / total);
  if (done < total) requestAnimationFrame(watchProgress);
}
requestAnimationFrame(watchProgress);
```

```javascript
// worker.js
let progress;

self.onmessage = ({ data: { type, progressBuffer, data } }) => {
  if (type !== "START") return;

  progress = new Int32Array(progressBuffer);
  Atomics.store(progress, 1, data.length); // set total

  data.forEach((item, i) => {
    processItem(item);
    Atomics.store(progress, 0, i + 1); // update progress atomically
  });

  self.postMessage({ type: "DONE" });
};
```

> **Security note:** `SharedArrayBuffer` requires [Cross-Origin Isolation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#security_requirements) headers (`Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp`) due to Spectre mitigations.

---

## 7. Worker Lifecycle and Memory

### Lifecycle

```
Created (new Worker('./w.js'))
  │
  ▼
Loading (script downloads and parses)
  │
  ▼
Running (executing messages)
  │  ← runs indefinitely while page is open
  ▼
Terminated
  ├── worker.terminate() called from main thread (immediate, no cleanup)
  ├── self.close() called from inside worker (graceful, pending work finishes)
  └── Page unloads (all workers terminated)
```

### Memory Implications

```javascript
// ❌ Worker created per operation — expensive and leaky
async function heavyOperation(data) {
  const worker = new Worker("./worker.js"); // new thread every time!
  return new Promise((resolve, reject) => {
    worker.onmessage = ({ data }) => {
      resolve(data);
      worker.terminate(); // terminate after each use
    };
    worker.onerror = reject;
    worker.postMessage(data);
  });
  // Creating/destroying threads is expensive — OS overhead per operation
}

// ✅ Long-lived worker — create once, reuse
const sharedWorker = new Worker("./worker.js");
// use workerClient pattern from Section 4
```

### Worker Memory Is Isolated

Workers have their own heap — they don't share memory with the main thread (except via SharedArrayBuffer). This means:

- Main thread GC doesn't affect worker memory
- Worker memory leaks are isolated to the worker
- Terminating a worker frees ALL its memory immediately

```javascript
// Terminating a worker is the nuclear option for memory cleanup
const worker = new Worker("./worker.js");
// ... use worker ...
worker.terminate(); // immediately frees ALL worker memory
```

---

## 8. Worker Pools — Production Pattern

For applications that need to run many parallel tasks, a **worker pool** maintains a fixed number of workers and queues tasks when all workers are busy.

```javascript
class WorkerPool {
  #workers = [];
  #queue = [];
  #idle = [];

  constructor(scriptUrl, size = navigator.hardwareConcurrency ?? 4) {
    for (let i = 0; i < size; i++) {
      const worker = new Worker(scriptUrl);
      worker.onmessage = (e) => this.#onMessage(worker, e);
      worker.onerror = (e) => this.#onError(worker, e);
      this.#workers.push(worker);
      this.#idle.push(worker);
    }
  }

  /**
   * Run a task on an available worker.
   * Returns a Promise that resolves with the worker's response.
   */
  run(payload, transferList = []) {
    return new Promise((resolve, reject) => {
      const task = { payload, transferList, resolve, reject };

      if (this.#idle.length > 0) {
        this.#dispatch(this.#idle.pop(), task);
      } else {
        this.#queue.push(task); // all workers busy — queue
      }
    });
  }

  #dispatch(worker, task) {
    worker._currentTask = task;
    worker.postMessage(task.payload, task.transferList);
  }

  #onMessage(worker, { data }) {
    const task = worker._currentTask;
    worker._currentTask = null;

    if (this.#queue.length > 0) {
      // Immediately dispatch next queued task
      this.#dispatch(worker, this.#queue.shift());
    } else {
      this.#idle.push(worker); // worker goes idle
    }

    task.resolve(data);
  }

  #onError(worker, error) {
    const task = worker._currentTask;
    worker._currentTask = null;
    this.#idle.push(worker);
    task?.reject(error);
  }

  /**
   * Run tasks in parallel with max concurrency = pool size.
   */
  async map(items, toPayload = (x) => x) {
    return Promise.all(items.map((item) => this.run(toPayload(item))));
  }

  terminate() {
    this.#workers.forEach((w) => w.terminate());
    this.#workers = [];
    this.#idle = [];
    // Reject all queued tasks
    while (this.#queue.length > 0) {
      this.#queue.shift().reject(new Error("Pool terminated"));
    }
  }

  get stats() {
    return {
      total: this.#workers.length,
      idle: this.#idle.length,
      busy: this.#workers.length - this.#idle.length,
      queued: this.#queue.length,
    };
  }
}

// Usage
const pool = new WorkerPool("./compute-worker.js", 4);

// Process 1000 items with 4 parallel workers
const results = await pool.map(items, (item) => ({
  type: "PROCESS",
  data: item,
}));

console.log(pool.stats); // { total: 4, idle: 4, busy: 0, queued: 0 }
```

---

## 9. Shared Workers

A `SharedWorker` can be connected to by multiple tabs, iframes, or windows from the **same origin**. This enables cross-tab shared state without a server.

```javascript
// main.js (in multiple tabs)
const shared = new SharedWorker("./shared-worker.js");

// Communication happens through a MessagePort
shared.port.onmessage = ({ data }) => {
  console.log("from shared worker:", data);
};

shared.port.start(); // must call start() for shared workers
shared.port.postMessage({ type: "HELLO", tabId: Math.random() });
```

```javascript
// shared-worker.js
const connections = new Set(); // track all connected ports

self.onconnect = ({ ports }) => {
  const port = ports[0];
  connections.add(port);

  port.onmessage = ({ data }) => {
    if (data.type === "BROADCAST") {
      // Broadcast to ALL connected tabs
      connections.forEach((p) => {
        if (p !== port) p.postMessage(data.payload);
      });
    }
  };

  // Clean up when tab closes (port disconnects)
  port.addEventListener("close", () => {
    connections.delete(port);
  });

  port.start();
};
```

### Use Cases for Shared Workers

- Cross-tab notification system (one WebSocket connection shared across all tabs)
- Shared authentication state
- Cross-tab real-time data sync
- Shared background task (e.g., one tab does data polling for all)

### Browser Support Note

Shared Workers are not supported in Safari on iOS. For cross-tab communication on all browsers, consider `BroadcastChannel` API as an alternative.

---

## 10. Service Workers (Overview)

Service Workers are a different kind of worker — they act as a **network proxy** sitting between the page and the network. Covered in depth in [`javascript-core/13-service-workers.md`](./13-service-workers.md). Key differences:

```
Feature              Dedicated Worker    Service Worker
─────────────────────────────────────────────────────────
Purpose              CPU offload         Network proxy/cache
Scope                One page            Entire origin
Lifecycle            Page-tied           Independent (survives page close)
Creation             new Worker()        navigator.serviceWorker.register()
DOM access           No                  No
Network intercept    No                  Yes (fetch event)
Push notifications   No                  Yes
Background sync      No                  Yes
```

---

## 11. What Workers Can and Cannot Do

### Workers CAN

```javascript
// ✅ All of these work inside a worker:

// JavaScript execution
const result = heavyComputation();

// Timers
setTimeout(() => {}, 1000);
setInterval(() => {}, 500);

// Network requests
const data = await fetch("/api/data").then((r) => r.json());

// WebSockets
const ws = new WebSocket("wss://example.com");

// IndexedDB
const db = await openDB("mydb", 1, {
  /* ... */
});

// Crypto API
const hash = await crypto.subtle.digest("SHA-256", buffer);

// Canvas (OffscreenCanvas)
const canvas = new OffscreenCanvas(800, 600);
const ctx = canvas.getContext("2d");

// Import other scripts
importScripts("./lib.js"); // classic workers
// import './lib.js'; // module workers

// Cache API
const cache = await caches.open("v1");

// postMessage to main thread
self.postMessage({ result });
```

### Workers CANNOT

```javascript
// ❌ These are NOT available in workers:

// DOM manipulation
document.getElementById("btn"); // ReferenceError: document is not defined
document.body.appendChild(el); // ReferenceError

// Window object
window.location.href; // ReferenceError: window is not defined
window.alert("hi"); // ReferenceError

// localStorage / sessionStorage
localStorage.setItem("key", "value"); // ReferenceError

// Parent page access
parent.document; // not available

// Note: Use IndexedDB for persistent storage in workers
```

---

## 12. Comlink — RPC Over Workers

[Comlink](https://github.com/GoogleChromeLabs/comlink) is a library that makes workers feel like regular async function calls — eliminating the manual postMessage/onmessage boilerplate.

```javascript
// worker.js
import * as Comlink from "comlink";

const api = {
  add(a, b) {
    return a + b;
  },

  async heavyCompute(data) {
    // simulate heavy work
    await new Promise((r) => setTimeout(r, 100));
    return data.map((x) => x * x);
  },

  fibonacci(n) {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  },
};

Comlink.expose(api);
```

```javascript
// main.js
import * as Comlink from "comlink";

const worker = new Worker("./worker.js", { type: "module" });
const api = Comlink.wrap(worker);

// Call worker functions as if they're local async functions!
const sum = await api.add(1, 2); // 3
const result = await api.heavyCompute([1, 2, 3]); // [1, 4, 9]
const fib = await api.fibonacci(40); // runs in worker — main thread free

console.log(sum, result, fib);
```

Comlink handles:

- Message ID management
- Promise wrapping/unwrapping
- Error propagation
- Transferable handling (via `Comlink.transfer()`)

---

## 13. Real-World Use Cases

### Use Case 1 — Large Dataset Processing

```javascript
// Process 500k rows without freezing the table UI
const pool = new WorkerPool("./data-worker.js", 4);

async function analyzeDataset(rows) {
  // Split into 4 chunks, process in parallel
  const chunkSize = Math.ceil(rows.length / 4);
  const chunks = Array.from({ length: 4 }, (_, i) =>
    rows.slice(i * chunkSize, (i + 1) * chunkSize),
  );

  const partialResults = await pool.map(chunks, (chunk) => ({
    type: "ANALYZE",
    data: chunk,
  }));

  // Merge partial results on main thread (fast operation)
  return mergeResults(partialResults);
}
```

### Use Case 2 — Real-Time Image/Video Processing

```javascript
// Apply filters to video frames in a worker at 30fps
const filterWorker = new Worker("./filter-worker.js");
const workerClient = new WorkerClient(filterWorker);

async function processVideoFrame(videoElement) {
  const canvas = new OffscreenCanvas(
    videoElement.videoWidth,
    videoElement.videoHeight,
  );
  const ctx = canvas.getContext("2d");
  ctx.drawImage(videoElement, 0, 0);

  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  const buffer = imageData.data.buffer;

  // Transfer pixel buffer to worker (zero-copy)
  const result = await workerClient.send(
    "APPLY_FILTER",
    { buffer, width: canvas.width, height: canvas.height, filter: "blur" },
    [buffer], // transfer list
  );

  // Put processed pixels back
  const processedData = new ImageData(
    new Uint8ClampedArray(result.buffer),
    result.width,
    result.height,
  );
  ctx.putImageData(processedData, 0, 0);
}
```

### Use Case 3 — JSON Parsing Large Payloads

```javascript
// Large JSON responses (> 1MB) can block main thread during parse
// Offload to a worker

// worker.js
self.onmessage = ({ data: { type, jsonString } }) => {
  if (type === "PARSE_JSON") {
    try {
      const parsed = JSON.parse(jsonString); // heavy parsing in worker
      self.postMessage({ type: "PARSED", result: parsed });
    } catch (err) {
      self.postMessage({ type: "PARSE_ERROR", error: err.message });
    }
  }
};

// main.js
async function fetchLargeDataset(url) {
  const response = await fetch(url);
  const jsonText = await response.text(); // get as string (fast)

  // Parse in worker (non-blocking)
  return workerClient.send("PARSE_JSON", { jsonString: jsonText });
}
```

### Use Case 4 — Topology/Graph Layout Calculation

```javascript
// Force-directed graph layout is O(n²) per frame — perfect for workers

// worker.js
self.onmessage = ({ data: { type, nodes, edges } }) => {
  if (type === "RUN_LAYOUT") {
    // Run 100 iterations of force-directed layout
    const result = forceDirectedLayout(nodes, edges, 100);
    self.postMessage({ type: "LAYOUT_DONE", nodes: result });
  }
};

// main.js: run layout without freezing the visualization
async function updateLayout(graph) {
  showLayoutSpinner();
  const updatedNodes = await workerClient.send("RUN_LAYOUT", {
    nodes: graph.nodes,
    edges: graph.edges,
  });
  graph.applyLayout(updatedNodes);
  hideLayoutSpinner();
}
```

---

## 14. Good Practices

### ✅ Use a worker pool sized to `hardwareConcurrency`

```javascript
const POOL_SIZE = navigator.hardwareConcurrency ?? 4;
// Don't create more workers than CPU cores — diminishing returns + overhead
const pool = new WorkerPool("./worker.js", POOL_SIZE);
```

### ✅ Transfer large buffers instead of copying

```javascript
// ✅ Transfer ArrayBuffer — zero-copy
worker.postMessage({ buffer: largeBuffer }, [largeBuffer]);

// ❌ Clone by default — expensive for large data
worker.postMessage({ buffer: largeBuffer }); // copies entire buffer
```

### ✅ Use TypedArrays for numeric data exchange

```javascript
// ✅ Float32Array for position data — dense, transferable, cache-friendly
const positions = new Float32Array(nodes.length * 2);
nodes.forEach((node, i) => {
  positions[i * 2] = node.x;
  positions[i * 2 + 1] = node.y;
});
worker.postMessage({ positions }, [positions.buffer]);
```

### ✅ Terminate workers when no longer needed

```javascript
// ✅ Clean up workers to free OS threads
class FeatureModule {
  #worker = new Worker("./worker.js");

  async compute(data) {
    return workerClient.send("COMPUTE", data);
  }

  destroy() {
    this.#worker.terminate();
  }
}
```

### ✅ Handle worker errors at the application level

```javascript
worker.onerror = (error) => {
  console.error(
    `Worker failed: ${error.message} in ${error.filename}:${error.lineno}`,
  );
  reportError(error);
  // Restart worker or degrade gracefully
  restartWorker();
};
```

---

## 15. Bad Practices

### ❌ Creating a new Worker for every task

```javascript
// ❌ OS thread creation overhead on every operation
async function processItem(item) {
  return new Promise((resolve) => {
    const w = new Worker("./worker.js");
    w.onmessage = (e) => {
      resolve(e.data);
      w.terminate();
    };
    w.postMessage(item);
  });
}
// With 100 items: 100 thread creations — very expensive
```

### ❌ Passing large objects by reference (they're copied anyway)

```javascript
// ❌ Misunderstanding: objects are NOT passed by reference to workers
const hugeGraph = buildGraph(); // 50MB object
worker.postMessage(hugeGraph); // creates a 50MB copy — 100MB total RAM
// hugeGraph is unmodified in main thread
```

### ❌ Using workers for trivial operations

```javascript
// ❌ Overhead of postMessage exceeds computation benefit
worker.postMessage({ type: "ADD", a: 1, b: 2 });
// IPC overhead >> time to just do `1 + 2` on main thread
```

### ❌ Not handling worker errors

```javascript
// ❌ Silently broken worker
const worker = new Worker("./worker.js");
worker.postMessage(data);
worker.onmessage = (e) => render(e.data);
// No onerror handler — if worker crashes, nothing happens
```

---

## 16. Interview-Level Explanation

> **"What are Web Workers? When would you use them? What are the limitations?"**

**Strong answer:**

> "Web Workers give JavaScript true parallelism by running JavaScript on separate OS threads, independent of the main thread. This is different from async/await, which is cooperative concurrency — async code still runs on the main thread and can block rendering. Workers run simultaneously on different CPU cores.
>
> You'd use workers for CPU-intensive tasks that would freeze the UI — large dataset processing, image/video manipulation, complex calculations like physics simulations or graph layouts, JSON parsing of large payloads, or cryptographic operations. The basic model is: send data to the worker via postMessage, worker processes it on a background thread, sends result back via postMessage.
>
> By default, data passed between threads is deep-copied using the Structured Clone algorithm — for large datasets this can be expensive. The solution is transferable objects: ArrayBuffers and other types can be transferred (not copied) using the transfer list in postMessage. Ownership moves to the receiving thread, the original is neutered — O(1) instead of O(n) data transfer.
>
> For true shared memory, SharedArrayBuffer lets both threads access the same memory, with Atomics providing thread-safe operations to prevent race conditions.
>
> The key limitations: workers can't access the DOM, window, or localStorage — they have their own isolated global scope. All communication is message-passing. And spawning workers has overhead — creating a new thread for every task is expensive. In production, you use a worker pool sized to navigator.hardwareConcurrency to amortize the thread creation cost across many tasks."

---

## 17. Exercises

### Exercise 1 — Offload a heavy loop

The following code freezes the UI. Refactor it to use a Web Worker:

```javascript
// ❌ Freezes UI for ~3 seconds
function findPrimes(limit) {
  const primes = [];
  for (let n = 2; n <= limit; n++) {
    let isPrime = true;
    for (let i = 2; i <= Math.sqrt(n); i++) {
      if (n % i === 0) {
        isPrime = false;
        break;
      }
    }
    if (isPrime) primes.push(n);
  }
  return primes;
}

const primes = findPrimes(10_000_000); // 3+ seconds
renderPrimes(primes);
```

<details>
<summary>Solution</summary>

```javascript
// worker.js
self.onmessage = ({ data: { limit } }) => {
  const primes = [];
  for (let n = 2; n <= limit; n++) {
    let isPrime = true;
    for (let i = 2; i <= Math.sqrt(n); i++) {
      if (n % i === 0) {
        isPrime = false;
        break;
      }
    }
    if (isPrime) primes.push(n);

    // Optional: send progress updates every 100k
    if (n % 100_000 === 0) {
      self.postMessage({ type: "PROGRESS", value: n / limit });
    }
  }
  self.postMessage({ type: "DONE", primes });
};

// main.js
const worker = new Worker("./worker.js");

worker.onmessage = ({ data }) => {
  if (data.type === "PROGRESS") {
    updateProgressBar(data.value);
  } else if (data.type === "DONE") {
    renderPrimes(data.primes);
    worker.terminate();
  }
};

worker.postMessage({ limit: 10_000_000 });
// Main thread remains responsive during computation
```

</details>

---

### Exercise 2 — Implement a worker pool

Using the `WorkerPool` class from Section 8 as a reference, implement a simplified version with:

- Fixed pool size
- Task queuing when all workers are busy
- `run(payload)` → Promise
- `terminate()` to shut down all workers

Test it by processing 20 tasks with a pool of 3 workers.

---

### Exercise 3 — Measure the transfer speedup

```javascript
// Run this benchmark comparing clone vs transfer for a 50MB buffer

const SIZE = 50 * 1024 * 1024; // 50MB

const worker = new Worker(
  URL.createObjectURL(
    new Blob(
      [
        `
  self.onmessage = (e) => self.postMessage('ok', e.data.buffer ? [e.data.buffer] : []);
`,
      ],
      { type: "application/javascript" },
    ),
  ),
);

// Test 1: Clone (default)
const buf1 = new ArrayBuffer(SIZE);
const t1 = performance.now();
worker.postMessage({ buffer: buf1 }); // clone — no transfer list
await new Promise((r) => (worker.onmessage = r));
console.log("Clone:", (performance.now() - t1).toFixed(1) + "ms");

// Test 2: Transfer
const buf2 = new ArrayBuffer(SIZE);
const t2 = performance.now();
worker.postMessage({ buffer: buf2 }, [buf2]); // transfer
await new Promise((r) => (worker.onmessage = r));
console.log("Transfer:", (performance.now() - t2).toFixed(1) + "ms");

// Expected: Clone ~50ms, Transfer ~0.1ms
```

---

## 🔗 Related Topics

- [`javascript-core/13-service-workers.md`](./13-service-workers.md) — Service workers for caching and offline
- [`javascript-core/03-event-loop.md`](./03-event-loop.md) — How the main thread event loop works
- [`rendering/03-cooperative-scheduling.md`](../rendering/03-cooperative-scheduling.md) — Alternatives to workers for long tasks
- [`performance/12-large-data-rendering.md`](../performance/12-large-data-rendering.md) — Processing large datasets

---

<div align="center">

**Next:** [`javascript-core/13-service-workers.md`](./13-service-workers.md) →

</div>
