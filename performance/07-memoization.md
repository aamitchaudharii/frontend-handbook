# 07 — Memoization

> **"Memoization is a trade: you spend memory to save time. Like all trades, it's only worth making when the time saved exceeds the cost of the memory spent — and only when the inputs actually repeat."**

Memoization caches the result of a function call and returns the cached result when the same inputs appear again. It's one of the oldest optimization techniques in computer science, and one of the most misapplied in frontend development. This document covers memoization from first principles: when it helps, when it hurts, correct implementation for all input types, cache eviction, and the React-specific patterns (useMemo, useCallback, memo) that apply these ideas in a framework context.

---

## 📚 Table of Contents

1. [What Memoization Is](#1-what-memoization-is)
2. [When Memoization Helps](#2-when-memoization-helps)
3. [When Memoization Hurts](#3-when-memoization-hurts)
4. [Basic Implementation](#4-basic-implementation)
5. [Multi-Argument Memoization](#5-multi-argument-memoization)
6. [Deep Equality Memoization](#6-deep-equality-memoization)
7. [LRU Cache — Bounded Memoization](#7-lru-cache--bounded-memoization)
8. [Memoizing Async Functions](#8-memoizing-async-functions)
9. [React.memo — Component Memoization](#9-reactmemo--component-memoization)
10. [useMemo — Value Memoization](#10-usememo--value-memoization)
11. [useCallback — Function Memoization](#11-usecallback--function-memoization)
12. [The React Memoization Decision Framework](#12-the-react-memoization-decision-framework)
13. [Selector Memoization (Reselect Pattern)](#13-selector-memoization-reselect-pattern)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. What Memoization Is

Memoization is a specific form of caching where a function stores its output, keyed by its inputs. On subsequent calls with the same inputs, the cached output is returned instead of recomputing.

```javascript
// Without memoization:
fibonacci(40); // computes ~330 million recursive calls → ~1 second
fibonacci(40); // computes ~330 million recursive calls again → ~1 second

// With memoization:
fibonacci(40); // computes once → ~1 second
fibonacci(40); // cache hit → ~0.001ms
```

### The Core Requirements

```
For memoization to be correct, the function must be:

1. PURE: same inputs always produce the same output
   - No side effects (no DOM mutations, no API calls)
   - No dependency on external mutable state
   - If f(x) can return different values for the same x: DO NOT memoize

2. DETERMINISTIC: output depends only on inputs
   - Math.random(), Date.now() in the function → not deterministic → don't memoize
   - Functions that read from a changing store → not deterministic
```

```javascript
// ✅ Safe to memoize: pure, deterministic
function expensive(arr, multiplier) {
  return arr.map((x) => x * multiplier).reduce((a, b) => a + b, 0);
}

// ❌ NOT safe to memoize: reads from external mutable state
function getFormattedTotal() {
  return formatCurrency(store.getState().cart.total); // state can change
}

// ❌ NOT safe to memoize: has side effects
function processAndLog(data) {
  const result = transform(data);
  analytics.track("processed", data); // side effect!
  return result;
}
```

---

## 2. When Memoization Helps

Memoization has measurable benefit when ALL of these are true:

```
1. The function is EXPENSIVE to compute
   (> ~1ms — if it's cheap, cache overhead may exceed the savings)

2. The same inputs appear REPEATEDLY
   (if inputs are always unique, cache never hits → wasted memory)

3. The function is PURE (same inputs → same outputs always)
   (incorrect for impure functions — may serve stale cached values)

4. The input set is BOUNDED
   (if inputs grow without limit, cache grows without limit → memory leak)
```

### Ideal Use Cases

```javascript
// ✅ Fibonacci: expensive recursion, same n called many times
const fib = memoize((n) => (n <= 1 ? n : fib(n - 1) + fib(n - 2)));

// ✅ Heavy data transformation: large array, same data fed repeatedly
const processedData = memoize((rawData) => expensiveTransform(rawData));

// ✅ Format functions called many times with repeated values
const formatDate = memoize((timestamp) =>
  new Intl.DateTimeFormat("en-US").format(timestamp),
);
// A table with 10,000 rows where many share the same date → 10,000 calls → few unique values

// ✅ Sorting/filtering: same filter criteria applied multiple times
const filterProducts = memoize((products, category) =>
  products.filter((p) => p.category === category),
);

// ✅ Selector computation in state management
const selectFilteredItems = memoize((items, filter) =>
  items.filter((item) => item.status === filter),
);
```

---

## 3. When Memoization Hurts

```javascript
// ❌ Cheap function: memoization overhead exceeds savings
const memoizedAdd = memoize((a, b) => a + b);
// Cache lookup + storage is slower than just computing a + b

// ❌ Always-unique inputs: cache never hits → just wastes memory
const memoizedCreateUser = memoize((userData) => createUser(userData));
// userData is always a new object with new values → cache grows, never hits

// ❌ Large inputs as cache keys: hashing or serializing them is expensive
const memoizedProcess = memoize((largeArray) => transform(largeArray));
// Hashing a 10,000-element array to create the cache key is expensive
// May be slower than just recomputing

// ❌ Unbounded cache: memory leak
const memoizedFetch = memoize((url) => fetch(url).then((r) => r.json()));
// Unique URLs accumulate indefinitely in the cache
// Solution: LRU cache with a size limit
```

---

## 4. Basic Implementation

### Single-Argument Memoization

```javascript
function memoize(fn) {
  const cache = new Map();

  return function memoized(arg) {
    if (cache.has(arg)) {
      return cache.get(arg);
    }

    const result = fn.call(this, arg);
    cache.set(arg, result);
    return result;
  };
}

// Usage
const expensiveSquareRoot = memoize((n) => {
  console.log("Computing...");
  return Math.sqrt(n);
});

expensiveSquareRoot(16); // "Computing..." → 4
expensiveSquareRoot(16); // (no log) → 4 (from cache)
expensiveSquareRoot(25); // "Computing..." → 5
```

### Cache Key Considerations

The default cache key is the first argument as-is. This works for primitives (numbers, strings, booleans) but NOT for objects:

```javascript
// ✅ Works: primitive keys (same value = same reference with Map)
memoize(fn)(42); // cached at key 42
memoize(fn)(42); // cache hit
memoize(fn)("hello"); // cached at key 'hello'

// ❌ Fails: object keys (different reference = cache miss even if equal)
const fn = memoize(processData);
fn({ filter: "active" }); // cached at reference A
fn({ filter: "active" }); // NEW reference B → cache miss!
// Two identical objects are different Map keys

// Solutions:
// 1. Use a string cache key
// 2. Use JSON.stringify as key (for simple objects)
// 3. Use deep equality cache (see Section 6)
// 4. Use a WeakMap (for object identity, no memory leak)
```

---

## 5. Multi-Argument Memoization

```javascript
function memoize(fn, { keyFn } = {}) {
  const cache = new Map();

  return function memoized(...args) {
    // Generate cache key from all arguments
    const key = keyFn
      ? keyFn(...args) // custom key function
      : args.length === 1
        ? args[0] // single-arg: use arg directly
        : JSON.stringify(args); // multi-arg: serialize to string

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Usage: default JSON.stringify key for primitives
const add = memoize((a, b) => a + b);
add(2, 3); // key: "[2,3]" → 5
add(2, 3); // cache hit → 5

// Usage: custom key function
const filterItems = memoize(
  (items, category, status) =>
    items.filter((i) => i.category === category && i.status === status),
  {
    // Key ignores `items` array reference — only uses category + status
    // (assumes items is always the same reference from the store)
    keyFn: (items, category, status) => `${category}:${status}`,
  },
);
```

### JSON.stringify Key Limitations

```javascript
// JSON.stringify is simple but has edge cases:
JSON.stringify([undefined, 1]); // "[null,1]" — undefined becomes null
JSON.stringify([fn, 1]); // "[null,1]" — functions become null
JSON.stringify({ b: 1, a: 2 }); // '{"b":1,"a":2}' — order matters!
JSON.stringify({ a: 2, b: 1 }); // '{"a":2,"b":1}' — different key!

// For objects where property order varies, normalize the key:
const stableKey = (obj) =>
  JSON.stringify(
    Object.fromEntries(
      Object.entries(obj).sort(([a], [b]) => a.localeCompare(b)),
    ),
  );
```

---

## 6. Deep Equality Memoization

When inputs are objects that aren't guaranteed to be the same reference, you need deep equality comparison:

```javascript
function deepEqual(a, b) {
  if (a === b) return true;
  if (a == null || b == null) return a === b;
  if (typeof a !== typeof b) return false;

  if (Array.isArray(a)) {
    if (!Array.isArray(b) || a.length !== b.length) return false;
    return a.every((v, i) => deepEqual(v, b[i]));
  }

  if (typeof a === "object") {
    const keysA = Object.keys(a);
    const keysB = Object.keys(b);
    if (keysA.length !== keysB.length) return false;
    return keysA.every((k) => deepEqual(a[k], b[k]));
  }

  return false;
}

function memoizeDeep(fn) {
  const cache = []; // array of { args, result } entries

  return function memoized(...args) {
    // Linear search for matching args (fine for small result sets)
    const entry = cache.find((e) => deepEqual(e.args, args));
    if (entry) return entry.result;

    const result = fn.apply(this, args);
    cache.push({ args, result });
    return result;
  };
}

// Usage: works even when object reference changes
const process = memoizeDeep((config) => heavyComputation(config));
process({ filter: "active", sort: "name" }); // computes
process({ filter: "active", sort: "name" }); // cache hit (deep equal)
```

---

## 7. LRU Cache — Bounded Memoization

An unbounded cache is a memory leak. An LRU (Least Recently Used) cache evicts the oldest-used entry when full.

```javascript
class LRUCache {
  #capacity;
  #cache; // Map preserves insertion order — we use this for LRU

  constructor(capacity) {
    this.#capacity = capacity;
    this.#cache = new Map();
  }

  get(key) {
    if (!this.#cache.has(key)) return undefined;

    // Move to end (most recently used)
    const value = this.#cache.get(key);
    this.#cache.delete(key);
    this.#cache.set(key, value);
    return value;
  }

  set(key, value) {
    if (this.#cache.has(key)) {
      this.#cache.delete(key); // remove to update position
    } else if (this.#cache.size >= this.#capacity) {
      // Evict LRU: first entry in Map (least recently used)
      const lruKey = this.#cache.keys().next().value;
      this.#cache.delete(lruKey);
    }

    this.#cache.set(key, value);
  }

  has(key) {
    return this.#cache.has(key);
  }
  get size() {
    return this.#cache.size;
  }
  clear() {
    this.#cache.clear();
  }
}

// Bounded memoize using LRU
function memoizeWithLRU(fn, { maxSize = 100, keyFn } = {}) {
  const cache = new LRUCache(maxSize);

  return function memoized(...args) {
    const key = keyFn ? keyFn(...args) : JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Memoize API response formatter with bounded cache
const formatApiResponse = memoizeWithLRU(
  (response) => expensiveTransform(response),
  { maxSize: 200 },
);
```

---

## 8. Memoizing Async Functions

```javascript
function memoizeAsync(fn, { maxSize = 100, ttlMs } = {}) {
  const cache = new LRUCache(maxSize);
  const pending = new Map(); // in-flight requests (deduplication)

  return async function memoizedAsync(...args) {
    const key = JSON.stringify(args);

    // Return cached result if fresh
    if (cache.has(key)) {
      const { value, timestamp } = cache.get(key);
      if (!ttlMs || Date.now() - timestamp < ttlMs) {
        return value;
      }
      cache.delete(key); // expired
    }

    // Deduplication: if same key is in flight, wait for it
    if (pending.has(key)) {
      return pending.get(key);
    }

    // Compute
    const promise = fn.apply(this, args).then(
      (result) => {
        cache.set(key, { value: result, timestamp: Date.now() });
        pending.delete(key);
        return result;
      },
      (error) => {
        pending.delete(key); // don't cache errors
        throw error;
      },
    );

    pending.set(key, promise);
    return promise;
  };
}

// Usage: memoized API fetch with 5-minute TTL
const fetchUser = memoizeAsync(
  async (userId) => {
    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  },
  { maxSize: 500, ttlMs: 5 * 60 * 1000 },
);

// Multiple concurrent calls with same userId → only ONE network request
await Promise.all([fetchUser("42"), fetchUser("42"), fetchUser("42")]);
// fetchUser('42') sends one request, all three await the same promise
```

---

## 9. React.memo — Component Memoization

`React.memo` wraps a component to prevent re-renders when props haven't changed (using shallow equality by default).

```javascript
// Without React.memo: re-renders whenever parent re-renders
function ExpensiveList({ items }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}

// With React.memo: only re-renders when `items` prop changes (shallow)
const ExpensiveList = React.memo(function ExpensiveList({ items }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
});
```

### Shallow vs Deep Comparison

```javascript
// React.memo uses SHALLOW comparison by default
// For primitive props: works correctly
// For object/array props: only checks reference equality

// ❌ Objects created inline: always new reference → memo never helps
function Parent() {
  return (
    <Child
      style={{ color: "red" }} // new object every render → Child always re-renders
      data={[1, 2, 3]} // new array every render
    />
  );
}

// ✅ Stable references: memo works
const STYLE = { color: "red" }; // outside component — stable reference
const DATA = [1, 2, 3];

function Parent() {
  return <Child style={STYLE} data={DATA} />; // same reference → memo works
}
```

### Custom Comparison in React.memo

```javascript
const ExpensiveList = React.memo(
  function ExpensiveList({ items, filter }) {
    return <ul>{/* ... */}</ul>;
  },
  (prevProps, nextProps) => {
    // Return true if props are "equal" (no re-render needed)
    // Return false to re-render
    return (
      prevProps.filter === nextProps.filter &&
      prevProps.items.length === nextProps.items.length &&
      prevProps.items.every((item, i) => item.id === nextProps.items[i].id)
    );
  },
);
```

---

## 10. useMemo — Value Memoization

`useMemo` memoizes a computed value within a component, recomputing only when dependencies change.

```javascript
function ProductList({ products, searchTerm, sortBy }) {
  // ❌ Recomputes on every render — even if products/searchTerm/sortBy haven't changed
  const filteredAndSorted = products
    .filter((p) => p.name.toLowerCase().includes(searchTerm.toLowerCase()))
    .sort((a, b) => a[sortBy].localeCompare(b[sortBy]));

  // ✅ Only recomputes when products, searchTerm, or sortBy changes
  const filteredAndSorted = useMemo(
    () =>
      products
        .filter((p) => p.name.toLowerCase().includes(searchTerm.toLowerCase()))
        .sort((a, b) => a[sortBy].localeCompare(b[sortBy])),
    [products, searchTerm, sortBy], // dependencies
  );

  return (
    <ul>
      {filteredAndSorted.map((p) => (
        <ProductCard key={p.id} product={p} />
      ))}
    </ul>
  );
}
```

### When useMemo Is Worth It

```javascript
// ✅ Worth it: expensive computation
const aggregatedStats = useMemo(
  () => computeExpensiveStats(largeDataset), // >1ms
  [largeDataset],
);

// ✅ Worth it: maintaining reference stability for child memo
const config = useMemo(
  () => ({ theme: userTheme, locale: userLocale }),
  [userTheme, userLocale],
);
// Without useMemo: new object every render → React.memo'd child re-renders anyway

// ❌ Not worth it: cheap computation
const doubled = useMemo(() => count * 2, [count]);
// Adding useMemo overhead is MORE expensive than just computing count * 2

// ❌ Not worth it: non-expensive and no stability needs
const title = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
// Simple string concatenation: useMemo overhead > computation cost
```

---

## 11. useCallback — Function Memoization

`useCallback` memoizes a function reference, returning the same function object across renders when dependencies haven't changed.

```javascript
function Parent({ items }) {
  // ❌ New function reference every render
  const handleDelete = (id) => {
    setItems((prev) => prev.filter((item) => item.id !== id));
  };

  // React.memo'd child receives new prop every render → always re-renders
  return <MemoizedItemList items={items} onDelete={handleDelete} />;
}

function Parent({ items }) {
  // ✅ Stable function reference across renders
  const handleDelete = useCallback(
    (id) => {
      setItems((prev) => prev.filter((item) => item.id !== id));
    },
    [], // empty deps: function never changes (uses functional update)
  );

  // MemoizedItemList only re-renders when items or handleDelete actually changes
  return <MemoizedItemList items={items} onDelete={handleDelete} />;
}
```

### useCallback Is Only Useful With React.memo

```javascript
// ❌ useCallback without React.memo — pointless overhead
const handleClick = useCallback(() => onClick(item.id), [item.id, onClick]);
// The child re-renders anyway (not memoized) — useCallback adds cost with zero benefit

// ✅ useCallback + React.memo — the combination that pays off
const MemoizedButton = React.memo(function Button({ onClick, label }) {
  return <button onClick={onClick}>{label}</button>;
});

function Parent({ item }) {
  const handleClick = useCallback(() => doSomething(item.id), [item.id]);
  return <MemoizedButton onClick={handleClick} label="Action" />;
  // handleClick is stable → MemoizedButton skips re-render
}
```

---

## 12. The React Memoization Decision Framework

Most developers over-memoize. Apply this framework before adding `React.memo`, `useMemo`, or `useCallback`:

```
BEFORE adding React.memo:
  1. Is this component actually slow? (Profile it)
  2. Does it re-render unnecessarily? (DevTools Profiler)
  3. Are ALL its props stable (primitives or memoized values)?
  4. If yes to all: add React.memo

BEFORE adding useMemo:
  1. Is this computation actually expensive? (>1ms)
  2. Are the inputs stable across renders that trigger this re-render?
  3. Does the output need to be referentially stable (fed to React.memo'd child)?
  4. If yes: add useMemo

BEFORE adding useCallback:
  1. Is this function passed to a React.memo'd child?
  2. OR is it in the dependency array of another hook that would misfires?
  3. If yes to either: add useCallback
  4. If no: skip it — useCallback adds overhead with no benefit

THE GOLDEN RULE:
  Measure first. Don't add memoization based on intuition.
  React DevTools Profiler shows exactly which components re-render
  and why — use it before and after any memoization change.
```

---

## 13. Selector Memoization (Reselect Pattern)

Selectors are functions that derive data from state. Memoizing them prevents expensive recomputations when unrelated state changes.

```javascript
// Without memoization: recomputes every time state changes (even unrelated parts)
const selectExpensiveStats = (state) => {
  return state.orders
    .filter((o) => o.status === "completed")
    .reduce(
      (acc, o) => ({
        count: acc.count + 1,
        revenue: acc.revenue + o.total,
      }),
      { count: 0, revenue: 0 },
    );
};

// Every state change (even unrelated) triggers this expensive computation
```

```javascript
// With Reselect pattern: memoized selector
import { createSelector } from "reselect"; // or implement manually

const selectCompletedOrders = (state) =>
  state.orders.filter((o) => o.status === "completed");

const selectOrderStats = createSelector(
  selectCompletedOrders, // input selector
  (completedOrders) => {
    // result function — only runs when completedOrders changes
    return completedOrders.reduce(
      (acc, o) => ({
        count: acc.count + 1,
        revenue: acc.revenue + o.total,
      }),
      { count: 0, revenue: 0 },
    );
  },
);

// selectOrderStats(state) only recomputes when state.orders changes
// Other state changes (UI, user, settings) → same memoized result
```

### Manual Selector Memoization

```javascript
function createSelector(...inputSelectors) {
  const resultFn = inputSelectors.pop(); // last arg is the result function

  let lastInputs = null;
  let lastResult = null;

  return function selector(state) {
    const inputs = inputSelectors.map((sel) => sel(state));

    // If all inputs are the same (by reference): return cached result
    if (
      lastInputs !== null &&
      inputs.length === lastInputs.length &&
      inputs.every((input, i) => input === lastInputs[i])
    ) {
      return lastResult;
    }

    lastInputs = inputs;
    lastResult = resultFn(...inputs);
    return lastResult;
  };
}
```

---

## 14. Good Practices

### ✅ Profile before memoizing

```javascript
// ✅ Measure the actual cost before adding memoization
console.time("filterProducts");
const result = products.filter((p) => p.category === "electronics");
console.timeEnd("filterProducts"); // 0.12ms for 1000 items — not worth memoizing

console.time("buildGraphData");
const graph = buildComplexGraph(nodes, edges); // 45ms for large datasets
console.timeEnd("buildGraphData"); // → worth memoizing
```

### ✅ Use WeakMap for object-keyed caches

```javascript
// WeakMap: object keys are held weakly — cache entries GC'd when object is GC'd
const cache = new WeakMap();

function processElement(element) {
  if (cache.has(element)) return cache.get(element);

  const result = expensiveProcessing(element);
  cache.set(element, result); // auto-cleaned when element is GC'd
  return result;
}
// No memory leak — cache entries disappear with their keys
```

### ✅ Always cap unbounded caches

```javascript
// ✅ Use LRU or TTL for any cache that grows with unique inputs
const processUrl = memoizeWithLRU(fetch, { maxSize: 100 });
// Max 100 entries — never grows unboundedly
```

### ✅ Make cache keys explicit and predictable

```javascript
// ✅ Clear about what the cache key is
const memoizedFormat = memoize(formatData, {
  keyFn: (data, options) =>
    `${options.locale}:${options.currency}:${data.amount}`,
});
// Cache key is the logical identity of the inputs
```

---

## 15. Bad Practices

### ❌ Memoizing impure functions

```javascript
// ❌ Reads from mutable external state — will serve stale results
const getFormattedTime = memoize(() => {
  return new Date().toLocaleTimeString(); // different every second
});
// First call at 10:30:00 → cached "10:30:00 AM"
// Next call at 10:30:01 → returns cached "10:30:00 AM" (wrong!)
```

### ❌ Memoizing without understanding the input types

```javascript
// ❌ Object inputs with default JSON.stringify key
const memoized = memoize((options) => compute(options));
memoized({ sort: "name", filter: "active" }); // key: '{"sort":"name","filter":"active"}'
memoized({ filter: "active", sort: "name" }); // key: '{"filter":"active","sort":"name"}' ← different!
// Same semantic input, different cache keys → cache miss
```

### ❌ Over-memoizing in React

```javascript
// ❌ Wrapping every component and computation in memo/useMemo
// This is common but rarely correct:
const SimpleText = React.memo(({ text }) => <span>{text}</span>);
const doubled = useMemo(() => count * 2, [count]);
const handler = useCallback(() => setOpen(true), []);

// React.memo's comparison function itself has overhead
// If the component renders anyway (props changed), you paid for memo for nothing
// Measure first — most components are fast enough without memo
```

---

## 16. Common Mistakes

### Mistake 1 — Memoize cache not shared across instances

```javascript
// ❌ Each component instance creates its own cache — no sharing
function Component({ id }) {
  const memoizedFetch = useMemo(() => memoize(fetchUser), []); // new cache per component!
  // If 100 components each memoize fetchUser with the same id:
  // 100 separate caches, 100 cache misses, 100 network requests
}

// ✅ Shared module-level cache
const fetchUserMemo = memoize(fetchUser); // defined once at module level

function Component({ id }) {
  const user = fetchUserMemo(id); // all instances share one cache
}
```

### Mistake 2 — Stale closures in memoized functions

```javascript
// ❌ Memoized function closes over stale variable
let multiplier = 1;
const memoizedMultiply = memoize((n) => n * multiplier);

memoizedMultiply(5); // 5 (multiplier = 1)
multiplier = 10;
memoizedMultiply(5); // 5 (from cache — uses stale multiplier!)
// Should be 50 but memoization cached the old result

// ✅ Make all dependencies explicit parameters
const memoizedMultiply = memoize((n, multiplier) => n * multiplier);
memoizedMultiply(5, 1); // 5, cached at key "[5,1]"
memoizedMultiply(5, 10); // 50, cached at key "[5,10]" (different key)
```

### Mistake 3 — useMemo dependency array errors

```javascript
// ❌ Missing dependency: stale closure
const result = useMemo(() => {
  return items.filter((item) => item.status === status); // status from outer scope
}, [items]); // forgot status! If status changes, result is stale

// ✅ Include all values used inside
const result = useMemo(() => {
  return items.filter((item) => item.status === status);
}, [items, status]);
```

### Mistake 4 — Caching error states

```javascript
// ❌ Caching errors permanently — transient errors stick
const memoizedFetch = memoize(async (url) => {
  const response = await fetch(url);
  if (!response.ok) throw new Error("Failed"); // cached! retry never works
  return response.json();
});

// ✅ Only cache successful results
const memoizedFetch = memoize(async (url) => {
  const response = await fetch(url);
  if (!response.ok) {
    // Don't let this throw reach the cache
    // (handled in memoizeAsync with pending cleanup)
    throw new Error("Failed");
  }
  return response.json();
}); // see memoizeAsync in Section 8 — errors delete from pending
```

---

## 17. Interview-Level Explanation

> **"What is memoization? When should you use it? How does React.memo differ from useMemo?"**

**Strong answer:**

> "Memoization caches the return value of a function, keyed by its inputs. On subsequent calls with the same inputs, the cached result is returned without re-executing the function. It's a time-memory tradeoff: you spend memory to save computation time.
>
> It's only correct for pure, deterministic functions — where the same inputs always produce the same outputs with no side effects. Memoizing a function that reads from mutable external state will serve stale results; memoizing one with side effects means effects only happen on first call.
>
> The three conditions for memoization to be worth it: the function is genuinely expensive (>1ms), the same inputs repeat frequently enough to hit the cache, and the input set is bounded so the cache doesn't grow forever. For unbounded inputs, use an LRU cache with a size limit.
>
> In React specifically: `React.memo` wraps a component to skip re-renders when props haven't changed — it does shallow prop comparison by default. `useMemo` memoizes a computed value within a component, recomputing only when specified dependencies change. `useCallback` memoizes a function reference to keep it stable across renders. These three work together: `useCallback` creates a stable function reference, which makes `React.memo` on the child component effective.
>
> The common mistake is over-memoizing. Most developers add `useMemo` and `useCallback` defensively everywhere, not realizing that the memoization mechanism itself has overhead — the comparison, the cache lookup, the storage. For a simple `count * 2`, `useMemo` costs more than just computing it. The rule is: profile first with React DevTools Profiler, then add memoization only where you can measure a meaningful improvement."

---

## 18. Exercises

### Exercise 1 — Implement memoize with LRU

Implement a `memoize` function that:

- Handles multi-argument functions
- Uses an LRU cache with configurable max size
- Allows a custom key function
- Returns the same memoized function that can be called normally

```javascript
const memoized = memoize(expensiveFunction, { maxSize: 50 });
memoized(1, 2, 3); // computes
memoized(1, 2, 3); // cache hit
```

<details>
<summary>Solution</summary>

```javascript
function memoize(fn, { maxSize = 100, keyFn } = {}) {
  // LRU cache using Map (insertion order)
  const cache = new Map();

  function getKey(args) {
    return keyFn ? keyFn(...args) : JSON.stringify(args);
  }

  function memoized(...args) {
    const key = getKey(args);

    if (cache.has(key)) {
      // Move to end (most recently used)
      const value = cache.get(key);
      cache.delete(key);
      cache.set(key, value);
      return value;
    }

    const result = fn.apply(this, args);

    // Evict LRU if at capacity
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    cache.set(key, result);
    return result;
  }

  memoized.cache = cache;
  memoized.clear = () => cache.clear();
  memoized.delete = (key) => cache.delete(key);
  memoized.original = fn;

  return memoized;
}

// Test:
const expensiveAdd = memoize(
  (a, b) => {
    console.log(`Computing ${a} + ${b}`);
    return a + b;
  },
  { maxSize: 3 },
);

expensiveAdd(1, 2); // Computing 1 + 2 → 3
expensiveAdd(1, 2); // (no log) → 3
expensiveAdd(3, 4); // Computing 3 + 4 → 7
expensiveAdd(5, 6); // Computing 5 + 6 → 11
expensiveAdd(7, 8); // Computing 7 + 8 (evicts [1,2]) → 15
expensiveAdd(1, 2); // Computing again (was evicted)
```

</details>

---

### Exercise 2 — Fix the React memoization

```jsx
// This component tree has memoization that isn't working correctly.
// Identify WHY it's not working and fix it.

const ItemList = React.memo(function ItemList({ items, onDelete }) {
  console.log("ItemList rendered");
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>
          {item.name}
          <button onClick={() => onDelete(item.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
});

function App() {
  const [items, setItems] = useState(initialItems);
  const [count, setCount] = useState(0);

  const handleDelete = (id) => {
    setItems((prev) => prev.filter((item) => item.id !== id));
  };

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <ItemList items={items} onDelete={handleDelete} />
    </div>
  );
}
```

<details>
<summary>Answer</summary>

```
Problem:
  handleDelete is created as a new function on every App render.
  Clicking "Count" increments count → App re-renders → new handleDelete reference.
  ItemList receives new onDelete prop → React.memo's shallow comparison fails.
  ItemList re-renders even though items hasn't changed.

Fix: wrap handleDelete in useCallback with stable dependencies

function App() {
  const [items, setItems] = useState(initialItems);
  const [count, setCount] = useState(0);

  // ✅ Stable function reference across renders
  const handleDelete = useCallback((id) => {
    setItems(prev => prev.filter(item => item.id !== id));
  }, []); // empty deps: uses functional updater, no stale closure

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ItemList items={items} onDelete={handleDelete} />
    </div>
  );
}

Now: Count increment → App re-renders → handleDelete is same reference
→ ItemList receives same items + same onDelete → React.memo skips re-render ✅
```

</details>

---

## 🔗 Related Topics

- [`performance/03-layout-thrashing.md`](./03-layout-thrashing.md) — What memoization helps avoid in rendering
- [`javascript-core/09-garbage-collection.md`](../javascript-core/09-garbage-collection.md) — Cache memory and GC
- [`javascript-core/05-closures.md`](../javascript-core/05-closures.md) — Closures in memoized functions
- [`patterns/03-command.md`](../patterns/03-command.md) — Command pattern with memoized selectors
- [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md) — Selector memoization in state management

---

<div align="center">

**Next:** [`performance/08-bundle-optimization.md`](./08-bundle-optimization.md) →

</div>
