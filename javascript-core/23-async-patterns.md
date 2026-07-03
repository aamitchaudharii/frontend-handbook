# 10 — Async Patterns

> **"Asynchronous code is not hard because of the syntax. It's hard because most developers never built a mental model for it. Once you have that model — callbacks, promises, async/await, and beyond — the patterns fall into place."**

JavaScript's async model is the foundation of every real-world frontend application. Network requests, timers, user interactions, animations — all async. This document goes beyond basic syntax to cover the full spectrum of async patterns, their tradeoffs, error handling strategies, composition techniques, cancellation, rate limiting, and production-level architecture.

---

## 📚 Table of Contents

1. [The Async Problem](#1-the-async-problem)
2. [Callbacks — The Foundation](#2-callbacks--the-foundation)
3. [Promises — Structured Async](#3-promises--structured-async)
4. [async/await — Sequential Style](#4-asyncawait--sequential-style)
5. [Error Handling Strategies](#5-error-handling-strategies)
6. [Parallel Execution Patterns](#6-parallel-execution-patterns)
7. [Sequential Execution Patterns](#7-sequential-execution-patterns)
8. [Cancellation — AbortController](#8-cancellation--abortcontroller)
9. [Timeouts and Deadlines](#9-timeouts-and-deadlines)
10. [Retry with Backoff](#10-retry-with-backoff)
11. [Rate Limiting and Throttling Async Operations](#11-rate-limiting-and-throttling-async-operations)
12. [Async Iterators and Generators](#12-async-iterators-and-generators)
13. [Debounce vs Throttle — Deep Dive](#13-debounce-vs-throttle--deep-dive)
14. [Deep Clone Optimization](#14-deep-clone-optimization)
15. [Mutation vs Immutability](#15-mutation-vs-immutability)
16. [Production Async Architecture](#16-production-async-architecture)
17. [Good Practices](#17-good-practices)
18. [Bad Practices](#18-bad-practices)
19. [Interview-Level Explanation](#19-interview-level-explanation)
20. [Exercises](#20-exercises)

---

## 1. The Async Problem

JavaScript runs on a single thread. When it needs to do something that takes time — fetch data, read a file, wait for a timer — it can't just block and wait. That would freeze the entire UI.

The solution: **schedule the work, yield to the event loop, resume when done.**

```
SYNCHRONOUS (blocks):
main() → fetchData() → [waiting 2 seconds...] → process() → done
         ↑ nothing else can run during waiting

ASYNCHRONOUS (non-blocking):
main() → fetchData() → [schedules request] → process other stuff
                                 ↓
                         [2 seconds later, response arrives]
                                 ↓
                         [callback/promise resolves] → process()
```

The challenge: expressing sequential logic (do A, then B, then C) in a system where each step may be deferred. Every async pattern in this document is a different approach to solving this same problem.

---

## 2. Callbacks — The Foundation

Callbacks are the most primitive async pattern. Every other pattern builds on them.

### Basic Callback Pattern

```javascript
// Node.js style: error-first callback
function readFile(path, callback) {
  // Simulate async operation
  setTimeout(() => {
    if (!path) {
      callback(new Error("No path provided"), null);
    } else {
      callback(null, `Contents of ${path}`);
    }
  }, 100);
}

readFile("./data.txt", (err, data) => {
  if (err) {
    console.error("Failed:", err.message);
    return;
  }
  console.log("Data:", data);
});
```

### Callback Hell — The Classic Problem

```javascript
// ❌ Deeply nested — hard to read, hard to handle errors, hard to maintain
getUser(userId, (err, user) => {
  if (err) return handleError(err);

  getPosts(user.id, (err, posts) => {
    if (err) return handleError(err);

    getComments(posts[0].id, (err, comments) => {
      if (err) return handleError(err);

      getAuthor(comments[0].authorId, (err, author) => {
        if (err) return handleError(err);

        render({ user, posts, comments, author });
        // Pyramid of doom — each level adds indentation
        // Error handling repeated at every level
        // Impossible to use try/catch
      });
    });
  });
});
```

### Flattening Callbacks — Named Functions

```javascript
// ✅ Named functions reduce nesting
function onAuthor(err, author) {
  if (err) return handleError(err);
  render({ user, posts, comments, author });
}

function onComments(err, comments) {
  if (err) return handleError(err);
  getAuthor(comments[0].authorId, onAuthor);
}

function onPosts(err, posts) {
  if (err) return handleError(err);
  getComments(posts[0].id, onComments);
}

function onUser(err, user) {
  if (err) return handleError(err);
  getPosts(user.id, onPosts);
}

getUser(userId, onUser);
// Still sequential, but flat — much more readable
```

### Promisifying Callbacks

```javascript
// Convert callback-based API to Promise-based
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}

const readFileAsync = promisify(readFile);
const data = await readFileAsync("./data.txt");
```

---

## 3. Promises — Structured Async

A Promise represents a value that will be available in the future. It can be in one of three states:

```
PENDING  → operation in progress
FULFILLED → operation succeeded (has a value)
REJECTED → operation failed (has a reason/error)

State transitions:
  PENDING → FULFILLED (resolve called)
  PENDING → REJECTED  (reject called)
  Once settled (fulfilled or rejected): IMMUTABLE — state never changes
```

### Promise Constructor

```javascript
function delay(ms) {
  return new Promise((resolve, reject) => {
    if (ms < 0) {
      reject(new Error("Delay must be non-negative"));
      return;
    }
    const id = setTimeout(resolve, ms);
    // Note: this Promise is not cancellable — we'll fix that later
  });
}

delay(1000)
  .then(() => console.log("1 second passed"))
  .catch((err) => console.error(err));
```

### Promise Chaining

Each `.then()` returns a **new Promise** — this is the key to flat, readable chains:

```javascript
fetchUser(userId)
  .then((user) => fetchPosts(user.id)) // returns new Promise
  .then((posts) => fetchComments(posts[0].id)) // chained
  .then((comments) => render(comments))
  .catch((err) => handleError(err)) // catches ANY error in the chain
  .finally(() => hideLoading()); // runs regardless of outcome
```

### How Promise Chaining Actually Works

```javascript
// Each .then() returns a NEW Promise
const p1 = fetchUser(userId);
// p1: Promise<User>

const p2 = p1.then((user) => fetchPosts(user.id));
// p2: Promise<Post[]> — resolves with what fetchPosts resolves with

const p3 = p2.then((posts) => posts.length);
// p3: Promise<number> — resolves with the posts count

// If any step throws or rejects:
const p4 = p1
  .then((user) => {
    throw new Error("fail");
  })
  .then(() => console.log("skipped")) // skipped
  .catch((err) => console.error(err.message)); // catches the throw
```

### Value Transformation in Chains

```javascript
// Returning a plain value from .then: wraps in resolved Promise
Promise.resolve(1)
  .then((n) => n + 1) // returns 2, wrapped as Promise.resolve(2)
  .then((n) => n * 10) // receives 2, returns 20
  .then((n) => console.log(n)); // 20

// Returning a Promise from .then: chain waits for it
Promise.resolve("url")
  .then((url) => fetch(url)) // returns Promise<Response> — chain WAITS
  .then((res) => res.json()) // receives Response once fetch resolves
  .then((data) => process(data));
```

---

## 4. async/await — Sequential Style

`async/await` is syntactic sugar over Promises. Every `async` function returns a Promise. Every `await` suspends the function and resumes it when the Promise settles.

### Basic async/await

```javascript
async function loadUserData(userId) {
  try {
    const user = await fetchUser(userId); // suspend until resolved
    const posts = await fetchPosts(user.id); // suspend until resolved
    const comments = await fetchComments(posts[0].id);

    return { user, posts, comments }; // returned value becomes Promise resolution
  } catch (err) {
    // catches any rejection in the await chain
    console.error("Failed to load:", err);
    throw err; // re-throw or return default
  }
}

// Usage:
const data = await loadUserData(42);
// or
loadUserData(42)
  .then((data) => render(data))
  .catch(handleError);
```

### async/await Desugaring

```javascript
// This async function:
async function example() {
  const a = await stepA();
  const b = await stepB(a);
  return b;
}

// Is equivalent to:
function example() {
  return stepA()
    .then((a) => stepB(a))
    .then((b) => b);
}
```

### Top-Level await (ES2022)

```javascript
// In ES modules — await at the top level of the module
// (not inside any function)
const config = await fetch("/config.json").then((r) => r.json());
const db = await connectDatabase(config.dbUrl);

export { db }; // module waits for db to be ready before exporting
```

### Common async/await Pitfalls

```javascript
// ❌ Sequential when parallel would be faster
async function loadBothSlowly() {
  const user = await fetchUser(1); // waits ~200ms
  const posts = await fetchPosts(1); // waits ~200ms AFTER user returns
  return { user, posts }; // total: ~400ms
}

// ✅ Parallel — both fire simultaneously
async function loadBothFast() {
  const [user, posts] = await Promise.all([
    fetchUser(1), // fires immediately
    fetchPosts(1), // fires immediately
  ]);
  return { user, posts }; // total: ~200ms (max of the two)
}
```

---

## 5. Error Handling Strategies

### Async Error Handling Patterns

```javascript
// Pattern 1: try/catch (most common with async/await)
async function safe() {
  try {
    const data = await riskyOperation();
    return { data, error: null };
  } catch (err) {
    return { data: null, error: err };
  }
}

// Pattern 2: Go-style tuple return (no throw propagation)
async function safeWrap(promise) {
  try {
    const data = await promise;
    return [null, data];
  } catch (err) {
    return [err, null];
  }
}

// Usage:
const [err, data] = await safeWrap(fetchUser(42));
if (err) {
  handleError(err);
  return;
}
render(data);
```

### Error Types — Distinguish and Handle

```javascript
class NetworkError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = "NetworkError";
    this.statusCode = statusCode;
  }
}

class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}

async function fetchUserSafe(id) {
  try {
    const res = await fetch(`/api/users/${id}`);

    if (!res.ok) {
      throw new NetworkError(`HTTP ${res.status}`, res.status);
    }

    const data = await res.json();

    if (!data.id) {
      throw new ValidationError("Missing user ID", "id");
    }

    return data;
  } catch (err) {
    if (err instanceof NetworkError) {
      if (err.statusCode === 404) return null; // not found — not an error
      if (err.statusCode === 401) redirectToLogin();
      throw err; // re-throw other network errors
    }
    if (err instanceof ValidationError) {
      logInvalidData(err);
      return null;
    }
    throw err; // unknown error — propagate
  }
}
```

### Unhandled Promise Rejections

```javascript
// Browser: listen for unhandled rejections globally
window.addEventListener("unhandledrejection", (event) => {
  console.error("Unhandled rejection:", event.reason);
  event.preventDefault(); // suppress default console error
  reportToErrorService(event.reason);
});

// Node.js:
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection at:", promise, "reason:", reason);
  process.exit(1); // fail fast in production
});

// ✅ Always handle Promise rejections:
fetchData()
  .then(process)
  .catch((err) => {
    // always have a .catch — never leave rejections unhandled
    handleError(err);
  });
```

---

## 6. Parallel Execution Patterns

### `Promise.all` — All or Nothing

```javascript
// All promises run in parallel
// Resolves when ALL resolve
// Rejects immediately if ANY rejects (others continue running but results ignored)
const [users, posts, config] = await Promise.all([
  fetchUsers(),
  fetchPosts(),
  fetchConfig(),
]);
```

### `Promise.allSettled` — Collect All Results

```javascript
// All promises run in parallel
// ALWAYS resolves (never rejects)
// Gives you the result of EACH, regardless of success/failure
const results = await Promise.allSettled([
  fetchUsers(),
  fetchPosts(),
  fetchConfig(),
]);

results.forEach((result, i) => {
  if (result.status === "fulfilled") {
    console.log(`Request ${i} succeeded:`, result.value);
  } else {
    console.error(`Request ${i} failed:`, result.reason);
  }
});
```

### `Promise.race` — First Wins

```javascript
// Resolves/rejects with the FIRST promise to settle (either way)
// Others continue running but their results are ignored
const result = await Promise.race([
  fetchData(),
  delay(5000).then(() => {
    throw new Error("Timeout");
  }),
]);
// If fetchData resolves in 2s and timeout is 5s → result = fetchData's value
// If fetchData takes 6s → throws TimeoutError
```

### `Promise.any` — First Success

```javascript
// Resolves with the FIRST promise to FULFILL
// Rejects only if ALL promises reject (AggregateError)
// Unlike Promise.race, ignores rejections until all have rejected
const result = await Promise.any([
  fetchFromServer1(), // might fail
  fetchFromServer2(), // might fail
  fetchFromServer3(), // at least one should succeed
]);
// Returns result from whichever server responds first successfully
```

### Comparison Table

| Method               | Resolves when  | Rejects when  | Use case                               |
| -------------------- | -------------- | ------------- | -------------------------------------- |
| `Promise.all`        | ALL fulfill    | ANY rejects   | All-or-nothing dependencies            |
| `Promise.allSettled` | ALL settle     | Never         | Collect all results, tolerate failures |
| `Promise.race`       | FIRST settles  | FIRST rejects | Timeout pattern, fastest of N          |
| `Promise.any`        | FIRST fulfills | ALL reject    | Fallback servers, redundancy           |

### Parallel with Concurrency Limit

```javascript
// ❌ No limit — fires ALL requests simultaneously
// For 1000 items: 1000 simultaneous network requests!
const results = await Promise.all(items.map(fetchItem));

// ✅ Limit concurrency to N simultaneous operations
async function parallelLimit(tasks, limit = 5) {
  const results = new Array(tasks.length);
  const queue = tasks.map((task, i) => ({ task, i }));
  const active = new Set();

  async function run({ task, i }) {
    const promise = task();
    active.add(promise);
    try {
      results[i] = await promise;
    } finally {
      active.delete(promise);
    }
  }

  // Process queue with max `limit` concurrent tasks
  const workers = Array.from({ length: limit }, async () => {
    while (queue.length > 0) {
      const item = queue.shift();
      if (item) await run(item);
    }
  });

  await Promise.all(workers);
  return results;
}

// Usage: fetch 1000 items, max 5 at once
const results = await parallelLimit(
  items.map((item) => () => fetchItem(item)),
  5,
);
```

---

## 7. Sequential Execution Patterns

### Sequential with `reduce`

```javascript
// Process array items in sequence, accumulating results
async function processSequentially(items) {
  return items.reduce(async (accPromise, item) => {
    const acc = await accPromise; // wait for previous
    const result = await processItem(item);
    return [...acc, result];
  }, Promise.resolve([]));
}
```

### Sequential with `for...of` (cleaner, more readable)

```javascript
// ✅ More readable than reduce for sequential async
async function processSequentially(items) {
  const results = [];
  for (const item of items) {
    const result = await processItem(item); // each waits for previous
    results.push(result);
  }
  return results;
}
```

### Pipeline Pattern

```javascript
// Process data through a sequence of async transforms
async function pipeline(initialValue, ...transforms) {
  let value = initialValue;
  for (const transform of transforms) {
    value = await transform(value);
  }
  return value;
}

// Usage
const result = await pipeline(rawData, validate, normalize, enrich, format);
```

---

## 8. Cancellation — AbortController

The `AbortController` API provides a standard way to cancel async operations.

### Basic Cancellation

```javascript
const controller = new AbortController();
const { signal } = controller;

async function fetchWithCancel(url, signal) {
  const response = await fetch(url, { signal });
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}

// Start fetch
const promise = fetchWithCancel("/api/data", signal);

// Cancel after 2 seconds
setTimeout(() => controller.abort(), 2000);

try {
  const data = await promise;
  render(data);
} catch (err) {
  if (err.name === "AbortError") {
    console.log("Fetch was cancelled");
  } else {
    throw err;
  }
}
```

### Cancellable Async Functions

```javascript
// Make any async function cancellable
function makeCancellable(asyncFn) {
  const controller = new AbortController();

  const promise = asyncFn(controller.signal).catch((err) => {
    if (err.name !== "AbortError") throw err;
    // AbortError is expected — don't propagate
  });

  return {
    promise,
    cancel: () => controller.abort(),
  };
}

// Usage
const { promise, cancel } = makeCancellable(async (signal) => {
  const data = await fetch("/api/data", { signal }).then((r) => r.json());
  return processData(data);
});

// Cancel if user navigates away
window.addEventListener("beforeunload", cancel);
```

### Cancel Previous Request (Search Pattern)

```javascript
// Classic search-as-you-type — cancel the previous request on each keystroke
class SearchService {
  #controller = null;

  async search(query) {
    // Cancel any in-flight request
    if (this.#controller) {
      this.#controller.abort();
    }

    // New controller for this request
    this.#controller = new AbortController();
    const { signal } = this.#controller;

    try {
      const results = await fetch(
        `/api/search?q=${encodeURIComponent(query)}`,
        { signal },
      ).then((r) => r.json());

      this.#controller = null;
      return results;
    } catch (err) {
      if (err.name === "AbortError") return null; // cancelled — ignore
      throw err;
    }
  }
}

const searchService = new SearchService();

input.addEventListener("input", async (e) => {
  const results = await searchService.search(e.target.value);
  if (results) renderResults(results); // null = was cancelled
});
```

### AbortSignal with Custom Async Operations

```javascript
// Use AbortSignal with non-fetch async operations
function delay(ms, signal) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) {
      reject(new DOMException("Aborted", "AbortError"));
      return;
    }

    const id = setTimeout(resolve, ms);

    signal?.addEventListener(
      "abort",
      () => {
        clearTimeout(id);
        reject(new DOMException("Aborted", "AbortError"));
      },
      { once: true },
    );
  });
}

// Cancellable polling
async function poll(url, interval, signal) {
  while (!signal.aborted) {
    const data = await fetch(url, { signal }).then((r) => r.json());
    process(data);
    await delay(interval, signal);
  }
}
```

---

## 9. Timeouts and Deadlines

### Promise Timeout Wrapper

```javascript
function withTimeout(promise, ms, message = "Operation timed out") {
  let timeoutId;

  const timeout = new Promise((_, reject) => {
    timeoutId = setTimeout(() => {
      reject(new Error(message));
    }, ms);
  });

  return Promise.race([promise, timeout]).finally(() =>
    clearTimeout(timeoutId),
  );
}

// Usage
const data = await withTimeout(
  fetchData(),
  5000,
  "fetchData timed out after 5s",
);
```

### AbortController-based Timeout

```javascript
// More robust: cancels the operation when timeout fires
function withTimeout(asyncFn, ms) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), ms);

  return asyncFn(controller.signal)
    .catch((err) => {
      if (err.name === "AbortError") {
        throw new Error(`Operation timed out after ${ms}ms`);
      }
      throw err;
    })
    .finally(() => clearTimeout(timeoutId));
}

// Usage
const data = await withTimeout(
  (signal) => fetch("/api/data", { signal }).then((r) => r.json()),
  5000,
);
```

### Deadline Pattern — Shared Timeout Across Multiple Operations

```javascript
class Deadline {
  constructor(ms) {
    this._controller = new AbortController();
    this._timeout = setTimeout(() => this._controller.abort(), ms);
    this.signal = this._controller.signal;
  }

  cancel() {
    clearTimeout(this._timeout);
    this._controller.abort();
  }

  get expired() {
    return this._controller.signal.aborted;
  }
}

// Usage: one deadline shared across multiple operations
async function loadPage(userId) {
  const deadline = new Deadline(10_000); // 10 second deadline for entire page load

  try {
    // All of these share the same deadline
    const [user, settings, feed] = await Promise.all([
      fetch(`/api/users/${userId}`, { signal: deadline.signal }).then((r) =>
        r.json(),
      ),
      fetch(`/api/settings`, { signal: deadline.signal }).then((r) => r.json()),
      fetch(`/api/feed`, { signal: deadline.signal }).then((r) => r.json()),
    ]);

    return { user, settings, feed };
  } catch (err) {
    if (err.name === "AbortError") {
      throw new Error("Page load timed out");
    }
    throw err;
  } finally {
    deadline.cancel(); // clear timeout if finished early
  }
}
```

---

## 10. Retry with Backoff

### Basic Retry

```javascript
async function retry(fn, maxAttempts = 3) {
  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      if (attempt < maxAttempts) {
        console.warn(`Attempt ${attempt} failed, retrying...`);
      }
    }
  }

  throw lastError;
}

// Usage
const data = await retry(() => fetchUser(42), 3);
```

### Exponential Backoff with Jitter

```javascript
/**
 * Retry with exponential backoff and optional jitter.
 * Delays: ~100ms, ~200ms, ~400ms, ~800ms... (doubles each time)
 * Jitter: adds randomness to prevent thundering herd
 */
async function retryWithBackoff(fn, options = {}) {
  const {
    maxAttempts = 3,
    baseDelayMs = 100,
    maxDelayMs = 30_000,
    jitter = true,
    shouldRetry = (err) => true, // retry all errors by default
    onRetry = null, // callback on each retry
    signal = null, // AbortSignal for cancellation
  } = options;

  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    if (signal?.aborted) {
      throw new DOMException("Aborted", "AbortError");
    }

    try {
      return await fn(attempt);
    } catch (err) {
      lastError = err;

      // Don't retry AbortErrors or if shouldRetry returns false
      if (err.name === "AbortError" || !shouldRetry(err, attempt)) {
        throw err;
      }

      if (attempt === maxAttempts) break; // last attempt — don't delay

      // Calculate delay: exponential backoff
      let delay = Math.min(baseDelayMs * Math.pow(2, attempt - 1), maxDelayMs);

      // Add jitter: random value between 50% and 100% of calculated delay
      if (jitter) {
        delay = delay * (0.5 + Math.random() * 0.5);
      }

      onRetry?.({ attempt, delay, error: err });

      await new Promise((resolve, reject) => {
        const id = setTimeout(resolve, delay);
        signal?.addEventListener(
          "abort",
          () => {
            clearTimeout(id);
            reject(new DOMException("Aborted", "AbortError"));
          },
          { once: true },
        );
      });
    }
  }

  throw lastError;
}

// Usage
const data = await retryWithBackoff(() => fetchUser(42), {
  maxAttempts: 5,
  baseDelayMs: 200,
  shouldRetry: (err) => err instanceof NetworkError && err.statusCode >= 500,
  onRetry: ({ attempt, delay }) =>
    console.log(`Retry ${attempt} in ${delay}ms`),
});
```

---

## 11. Rate Limiting and Throttling Async Operations

### Request Queue with Concurrency Limit

```javascript
class RequestQueue {
  constructor(concurrency = 5) {
    this._concurrency = concurrency;
    this._running = 0;
    this._queue = [];
  }

  add(fn) {
    return new Promise((resolve, reject) => {
      this._queue.push({ fn, resolve, reject });
      this._process();
    });
  }

  _process() {
    while (this._running < this._concurrency && this._queue.length > 0) {
      const { fn, resolve, reject } = this._queue.shift();
      this._running++;

      fn()
        .then(resolve)
        .catch(reject)
        .finally(() => {
          this._running--;
          this._process(); // trigger next item
        });
    }
  }

  get pending() {
    return this._queue.length;
  }
  get active() {
    return this._running;
  }
}

// Usage
const queue = new RequestQueue(3); // max 3 concurrent

const results = await Promise.all(
  items.map((item) => queue.add(() => fetchItem(item))),
);
```

### Token Bucket Rate Limiter

```javascript
// Limit to N requests per second
class TokenBucket {
  constructor(tokensPerSecond, maxBurst = tokensPerSecond) {
    this._tokensPerMs = tokensPerSecond / 1000;
    this._maxTokens = maxBurst;
    this._tokens = maxBurst;
    this._lastRefill = Date.now();
    this._queue = [];
  }

  _refill() {
    const now = Date.now();
    const elapsed = now - this._lastRefill;
    this._tokens = Math.min(
      this._maxTokens,
      this._tokens + elapsed * this._tokensPerMs,
    );
    this._lastRefill = now;
  }

  async consume(tokens = 1) {
    this._refill();

    if (this._tokens >= tokens) {
      this._tokens -= tokens;
      return; // immediately available
    }

    // Need to wait for tokens to refill
    const deficit = tokens - this._tokens;
    const waitMs = Math.ceil(deficit / this._tokensPerMs);

    await new Promise((resolve) => setTimeout(resolve, waitMs));
    this._refill();
    this._tokens -= tokens;
  }

  async wrap(fn) {
    await this.consume();
    return fn();
  }
}

// Usage: max 10 requests/second
const bucket = new TokenBucket(10);

async function fetchRateLimited(url) {
  await bucket.consume();
  return fetch(url).then((r) => r.json());
}
```

---

## 12. Async Iterators and Generators

### Async Generator — Lazy Async Sequences

```javascript
// Async generator: produces values on demand, asynchronously
async function* paginate(url, pageSize = 20) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${url}?page=${page}&size=${pageSize}`).then(
      (r) => r.json(),
    );

    yield* response.items; // yield each item from the page

    hasMore = response.hasNextPage;
    page++;
  }
}

// Usage: process users without loading all pages into memory
for await (const user of paginate("/api/users")) {
  await processUser(user); // process one at a time
  // Each `await` here pauses the async for loop
  // The generator fetches the next page lazily (only when needed)
}
```

### Async Iterator for Real-Time Data

```javascript
// AsyncIterator for WebSocket messages
function webSocketMessages(url) {
  let resolve;
  let reject;
  const queue = [];

  const ws = new WebSocket(url);

  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (resolve) {
      resolve({ value: data, done: false });
      resolve = null;
    } else {
      queue.push(data);
    }
  };

  ws.onerror = (err) => {
    if (reject) reject(err);
  };

  ws.onclose = () => {
    if (resolve) resolve({ value: undefined, done: true });
  };

  return {
    [Symbol.asyncIterator]() {
      return {
        next() {
          if (queue.length > 0) {
            return Promise.resolve({ value: queue.shift(), done: false });
          }
          return new Promise((res, rej) => {
            resolve = res;
            reject = rej;
          });
        },
        return() {
          ws.close();
          return Promise.resolve({ value: undefined, done: true });
        },
      };
    },
  };
}

// Usage
const stream = webSocketMessages("wss://api.example.com/live");
for await (const event of stream) {
  handleEvent(event);
}
```

---

## 13. Debounce vs Throttle — Deep Dive

Both control how frequently a function executes. They solve different problems.

### Debounce — Execute After Silence

```
Debounce: wait until input STOPS for `delay` ms, then execute ONCE

Events:  ↑  ↑  ↑  ↑  ↑ .... (delay) .... →  EXECUTE
         └──┘  └──┘  └─ start of silence period
         (each keystroke resets the timer)

Use cases: search-as-you-type, window resize handler, form validation
```

```javascript
function debounce(fn, delay) {
  let timerId = null;

  function debounced(...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => {
      timerId = null;
      fn.apply(this, args);
    }, delay);
  }

  // Cancel any pending execution
  debounced.cancel = () => {
    clearTimeout(timerId);
    timerId = null;
  };

  // Execute immediately and reset timer
  debounced.flush = function (...args) {
    clearTimeout(timerId);
    timerId = null;
    fn.apply(this, args);
  };

  return debounced;
}
```

### Throttle — Execute at Most Once Per Interval

```
Throttle: execute immediately, then wait `limit` ms before executing again

Events:  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑
         EXEC              EXEC              EXEC
         └─── limit ───┘   └─── limit ───┘

Use cases: scroll handlers, mousemove, button spam prevention
```

```javascript
function throttle(fn, limit) {
  let lastCall = 0;
  let timerId = null;

  function throttled(...args) {
    const now = Date.now();
    const remaining = limit - (now - lastCall);

    if (remaining <= 0) {
      // Enough time has passed — execute immediately
      if (timerId) {
        clearTimeout(timerId);
        timerId = null;
      }
      lastCall = now;
      fn.apply(this, args);
    } else if (!timerId) {
      // Schedule trailing call at end of throttle window
      timerId = setTimeout(() => {
        lastCall = Date.now();
        timerId = null;
        fn.apply(this, args);
      }, remaining);
    }
  }

  throttled.cancel = () => {
    clearTimeout(timerId);
    timerId = null;
    lastCall = 0;
  };

  return throttled;
}
```

### Key Difference

```javascript
const searchInput = document.getElementById("search");

// DEBOUNCE: fires ONCE after user stops typing for 300ms
// Good for: search — wait until user is done typing
searchInput.addEventListener("input", debounce(search, 300));

// THROTTLE: fires at most once per 300ms while user types
// Good for: autocomplete dropdown — update at 300ms intervals while typing
searchInput.addEventListener("input", throttle(autocomplete, 300));
```

---

## 14. Deep Clone Optimization

### Benchmarking Clone Strategies

```javascript
const testObj = {
  user: { name: "Alice", age: 30, address: { city: "NY" } },
  items: Array.from({ length: 1000 }, (_, i) => ({ id: i, value: i * 2 })),
  metadata: { created: new Date(), tags: ["a", "b", "c"] },
};

// Strategy 1: JSON round-trip (most common, has limitations)
function jsonClone(obj) {
  return JSON.parse(JSON.stringify(obj));
}
// Limitations: loses Date, undefined, functions, circular refs (throws)

// Strategy 2: structuredClone (modern, recommended)
function nativeClone(obj) {
  return structuredClone(obj);
}
// Handles: Date, Map, Set, ArrayBuffer, circular refs
// Doesn't handle: functions, DOM nodes, class instances (strips methods)

// Strategy 3: Recursive clone (full control)
function deepClone(obj, seen = new Map()) {
  if (obj === null || typeof obj !== "object") return obj;
  if (obj instanceof Date) return new Date(obj.getTime());
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);
  if (obj instanceof Map) {
    const map = new Map();
    seen.set(obj, map);
    obj.forEach((v, k) => map.set(deepClone(k, seen), deepClone(v, seen)));
    return map;
  }
  if (obj instanceof Set) {
    const set = new Set();
    seen.set(obj, set);
    obj.forEach((v) => set.add(deepClone(v, seen)));
    return set;
  }

  // Handle circular references
  if (seen.has(obj)) return seen.get(obj);

  const clone = Array.isArray(obj)
    ? []
    : Object.create(Object.getPrototypeOf(obj));
  seen.set(obj, clone);

  for (const key of Reflect.ownKeys(obj)) {
    clone[key] = deepClone(obj[key], seen);
  }

  return clone;
}

// Performance (approximate, varies by engine and data):
// JSON round-trip:    fastest for simple data, loses types
// structuredClone:    ~2-3x slower than JSON, handles more types
// Recursive clone:    slowest but most complete
```

### When to Use Each

```javascript
// Use JSON.parse(JSON.stringify()) when:
// - Data is simple (primitives, plain objects, arrays)
// - No Date, Map, Set, undefined, functions
// - No circular references
// - Maximum performance matters
const simpleClone = JSON.parse(JSON.stringify(simpleData));

// Use structuredClone when:
// - Modern browser/Node.js target
// - Need to handle Date, Map, Set, TypedArray, ArrayBuffer
// - Circular references possible
// - Most common "correct" choice
const robustClone = structuredClone(complexData);

// Use recursive/custom clone when:
// - Need to clone class instances (with methods)
// - Need special handling for custom types
// - structuredClone doesn't fit your needs
const fullClone = deepClone(instanceData);
```

---

## 15. Mutation vs Immutability

### The Case for Immutability

```javascript
// ❌ Mutation: state changes are hard to track
const state = { count: 0, items: ["a", "b"] };

function addItem(item) {
  state.items.push(item); // mutates original — who knows this changed?
  state.count++;
}

// ✅ Immutability: each change produces a new object
function addItem(state, item) {
  return {
    ...state,
    items: [...state.items, item], // new array
    count: state.count + 1,
  };
}
// Original state unchanged — easy to compare, debug, time-travel
```

### Efficient Immutable Updates — Structural Sharing

```javascript
// Full deep clone on every update is wasteful
// Structural sharing: only clone what changed

const before = {
  user: { name: "Alice", age: 30 }, // unchanged — reuse reference
  settings: { theme: "dark" }, // changed — new object
  items: ["a", "b", "c"], // unchanged — reuse reference
};

// Only the settings node is new — rest is shared
const after = {
  ...before, // spread: reuses user and items refs
  settings: { ...before.settings, theme: "light" }, // only this is new
};

// before.user === after.user  → true (same reference)
// before.settings === after.settings → false (new object)
```

### Immer-style Immutability (Draft Pattern)

```javascript
// Draft: write imperative mutation code, get immutable result
function produce(base, recipe) {
  // Simplified Immer-like implementation
  const draft = structuredClone(base); // start with a copy
  recipe(draft); // let recipe mutate the copy freely
  return Object.freeze(draft); // freeze and return
}

// Usage: write mutations, get immutable update
const nextState = produce(state, (draft) => {
  draft.user.name = "Bob"; // looks like mutation
  draft.items.push("d"); // looks like mutation
  draft.count++;
});
// state is unchanged; nextState is a new frozen object
```

---

## 16. Production Async Architecture

### Async State Machine

```javascript
// Model complex async flows as state machines
class RequestStateMachine {
  #state = "idle"; // idle | loading | success | error
  #data = null;
  #error = null;
  #controller = null;
  #listeners = new Set();

  get state() {
    return this.#state;
  }
  get data() {
    return this.#data;
  }
  get error() {
    return this.#error;
  }

  subscribe(fn) {
    this.#listeners.add(fn);
    return () => this.#listeners.delete(fn);
  }

  #notify() {
    this.#listeners.forEach((fn) =>
      fn({
        state: this.#state,
        data: this.#data,
        error: this.#error,
      }),
    );
  }

  async fetch(url) {
    // Cancel any in-flight request
    this.#controller?.abort();
    this.#controller = new AbortController();

    this.#state = "loading";
    this.#data = null;
    this.#error = null;
    this.#notify();

    try {
      const data = await fetch(url, { signal: this.#controller.signal }).then(
        (r) => {
          if (!r.ok) throw new Error(`HTTP ${r.status}`);
          return r.json();
        },
      );

      this.#state = "success";
      this.#data = data;
    } catch (err) {
      if (err.name === "AbortError") return; // cancelled — no state change
      this.#state = "error";
      this.#error = err;
    } finally {
      this.#controller = null;
      this.#notify();
    }
  }

  cancel() {
    this.#controller?.abort();
  }
}
```

---

## 17. Good Practices

### ✅ Always handle Promise rejections

```javascript
// ✅ Every Promise chain has a .catch
fetch("/api/data")
  .then((r) => r.json())
  .then(process)
  .catch((err) => handleError(err));

// ✅ Every async function has try/catch
async function loadData() {
  try {
    return await fetchData();
  } catch (err) {
    reportError(err);
    return null;
  }
}
```

### ✅ Use `Promise.allSettled` when partial failure is acceptable

```javascript
// ✅ Fetch all, handle each result independently
const [userResult, postsResult, adsResult] = await Promise.allSettled([
  fetchUser(),
  fetchPosts(),
  fetchAds(), // non-critical — OK if this fails
]);

const user = userResult.status === "fulfilled" ? userResult.value : null;
const posts = postsResult.status === "fulfilled" ? postsResult.value : [];
// Page renders even if some requests fail
```

### ✅ Cancel requests when they're no longer needed

```javascript
// ✅ Cancel on component unmount / route change
class DataFetcher {
  #controller = null;

  async fetch(url) {
    this.#controller?.abort();
    this.#controller = new AbortController();
    const data = await fetch(url, { signal: this.#controller.signal });
    return data.json();
  }

  destroy() {
    this.#controller?.abort();
  }
}
```

### ✅ Avoid `await` in loops for parallel operations

```javascript
// ❌ Sequential — slow
for (const id of ids) {
  users.push(await fetchUser(id));
}

// ✅ Parallel — fast
const users = await Promise.all(ids.map((id) => fetchUser(id)));
```

---

## 18. Bad Practices

### ❌ Unhandled Promise rejections

```javascript
// ❌ Rejection silently swallowed — no error logged, no user feedback
fetchData().then(process); // if fetchData rejects, nothing happens
```

### ❌ Creating Promises unnecessarily

```javascript
// ❌ Unnecessary Promise wrapping
async function getUser(id) {
  return new Promise((resolve) => {
    // pointless wrapper
    resolve(fetchUser(id)); // fetchUser already returns a Promise
  });
}

// ✅ Just return the Promise directly
async function getUser(id) {
  return fetchUser(id);
}
```

### ❌ Floating Promises in async functions

```javascript
// ❌ Fire-and-forget without error handling
async function process() {
  saveToDatabase(data); // not awaited, not .catch'd
  // If this rejects: unhandled rejection
  return "done";
}

// ✅ Always await or explicitly handle
async function process() {
  await saveToDatabase(data).catch((err) => logError(err));
  return "done";
}
```

### ❌ Sequential await when parallel is possible

```javascript
// ❌ Slow — 3 round trips in series
async function loadDashboard() {
  const user = await fetchUser(); // 200ms
  const metrics = await fetchMetrics(); // 200ms
  const alerts = await fetchAlerts(); // 200ms
  // Total: ~600ms
}

// ✅ Fast — all 3 in parallel
async function loadDashboard() {
  const [user, metrics, alerts] = await Promise.all([
    fetchUser(),
    fetchMetrics(),
    fetchAlerts(),
  ]); // Total: ~200ms
}
```

---

## 19. Interview-Level Explanation

> **"What is the difference between Promise.all, Promise.allSettled, Promise.race, and Promise.any? When would you use each?"**

**Strong answer:**

> "All four run their input promises in parallel — the difference is how they handle success and failure.
>
> `Promise.all` resolves when ALL promises fulfill, and rejects immediately if ANY reject. It's all-or-nothing. Use it when you need all results and can't proceed without them — loading user data that all depends on each other, for example.
>
> `Promise.allSettled` always resolves, giving you an array with each promise's outcome as either `{ status: 'fulfilled', value }` or `{ status: 'rejected', reason }`. Use it when partial failure is acceptable — rendering a dashboard where some widgets can fail without breaking the whole page.
>
> `Promise.race` resolves or rejects with the first promise to settle, regardless of success or failure. Classic use case: a timeout pattern — race your fetch against a delay promise that rejects after 5 seconds.
>
> `Promise.any` resolves with the first promise to FULFILL, ignoring rejections unless all reject (in which case it throws an AggregateError). Use it for redundancy — fire requests to multiple servers and use whichever responds first successfully.
>
> In production, `Promise.allSettled` is the safest default for loading multiple independent resources. `Promise.all` for dependent resources. `Promise.race` for timeouts. `Promise.any` for fallback servers or CDN failover."

---

## 20. Exercises

### Exercise 1 — Predict the output

```javascript
async function main() {
  console.log("1");

  const p = new Promise((resolve) => {
    console.log("2");
    resolve("3");
  });

  console.log("4");

  const result = await p;
  console.log(result);

  console.log("5");
}

main();
console.log("6");
```

<details>
<summary>Answer</summary>

```
1  — sync inside main
2  — Promise constructor runs synchronously
4  — sync inside main (before await)
6  — sync after main() call (main suspended at await)
3  — microtask: promise resolved, await resumes
5  — sync inside main (after await)

Output: 1, 2, 4, 6, 3, 5
```

</details>

---

### Exercise 2 — Implement `Promise.all` from scratch

```javascript
function myPromiseAll(promises) {
  // Implement the behavior of Promise.all
  // - Resolves with array of all values when all fulfill
  // - Rejects immediately when any reject
  // - Preserves order of results
}

// Test:
myPromiseAll([Promise.resolve(1), Promise.resolve(2), Promise.resolve(3)]).then(
  console.log,
); // [1, 2, 3]

myPromiseAll([
  Promise.resolve(1),
  Promise.reject(new Error("fail")),
  Promise.resolve(3),
]).catch((err) => console.error(err.message)); // 'fail'
```

<details>
<summary>Solution</summary>

```javascript
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      resolve([]);
      return;
    }

    const results = new Array(promises.length);
    let resolved = 0;

    promises.forEach((promise, i) => {
      Promise.resolve(promise)
        .then((value) => {
          results[i] = value;
          resolved++;

          if (resolved === promises.length) {
            resolve(results);
          }
        })
        .catch(reject); // first rejection wins
    });
  });
}
```

</details>

---

### Exercise 3 — Build a robust data fetcher

Implement a `DataFetcher` class that:

- Supports cancellation via AbortController
- Automatically retries on network errors (max 3 times, exponential backoff)
- Has a configurable timeout per request
- Returns `null` if cancelled (not an error)
- Throws descriptive errors for non-retriable failures (404, 401, etc.)

<details>
<summary>Solution outline</summary>

```javascript
class DataFetcher {
  #activeController = null;

  async fetch(url, options = {}) {
    const { timeout = 10_000, maxRetries = 3 } = options;

    // Cancel any previous request
    this.#activeController?.abort();
    this.#activeController = new AbortController();
    const { signal } = this.#activeController;

    try {
      return await retryWithBackoff(
        async () => {
          const timeoutController = new AbortController();
          const timeoutId = setTimeout(
            () => timeoutController.abort(),
            timeout,
          );

          // Combine external signal + timeout signal
          const combined = AbortSignal.any([signal, timeoutController.signal]);

          try {
            const res = await fetch(url, { signal: combined });
            clearTimeout(timeoutId);

            if (res.status === 404) return null; // not found — not retriable
            if (res.status === 401)
              throw Object.assign(new Error("Unauthorized"), {
                status: 401,
                retriable: false,
              });
            if (!res.ok)
              throw Object.assign(new Error(`HTTP ${res.status}`), {
                status: res.status,
                retriable: true,
              });

            return res.json();
          } finally {
            clearTimeout(timeoutId);
          }
        },
        {
          maxAttempts: maxRetries,
          baseDelayMs: 200,
          shouldRetry: (err) =>
            err.retriable !== false && err.name !== "AbortError",
          signal,
        },
      );
    } catch (err) {
      if (err.name === "AbortError") return null; // cancelled
      throw err;
    } finally {
      this.#activeController = null;
    }
  }

  cancel() {
    this.#activeController?.abort();
  }
}
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/03-event-loop.md`](./03-event-loop.md) — How async code is scheduled
- [`javascript-core/04-microtask-vs-macrotask.md`](./04-microtask-vs-macrotask.md) — Promise vs setTimeout timing
- [`javascript-core/11-promise-internals.md`](./11-promise-internals.md) — How Promises work internally
- [`javascript-core/12-web-workers.md`](./12-web-workers.md) — True parallelism for CPU-heavy tasks
- [`networking/03-request-batching.md`](../networking/03-request-batching.md) — Batching network requests

---

<div align="center">

**Next:** [`javascript-core/11-promise-internals.md`](./11-promise-internals.md) →

</div>
