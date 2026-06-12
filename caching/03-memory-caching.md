# 03 — Memory Caching

> **"Memory caching is trading RAM for CPU time. The question isn't whether to cache — it's what to cache, how long to keep it, how to invalidate it, and how to prevent the cache from consuming more memory than the work it saves."**

Memory caching — storing computed values in JavaScript data structures for immediate reuse — is the simplest and fastest form of caching available to frontend code. There's no network, no disk, no serialization: a cache hit is a Map lookup, typically under a microsecond. This document covers memoization, LRU caches, timed caches, React-specific memoization patterns, selector caches, and the memory management discipline needed to keep in-memory caches safe and bounded.

---

## 📚 Table of Contents

1. [What Memory Caching Is](#1-what-memory-caching-is)
2. [Memoization — Function Result Caching](#2-memoization--function-result-caching)
3. [LRU Cache — Bounded Eviction](#3-lru-cache--bounded-eviction)
4. [TTL Cache — Time-Based Expiration](#4-ttl-cache--time-based-expiration)
5. [LFU Cache — Frequency-Based Eviction](#5-lfu-cache--frequency-based-eviction)
6. [WeakMap Caching — GC-Friendly](#6-weakmap-caching--gc-friendly)
7. [React useMemo and useCallback](#7-react-usememo-and-usecallback)
8. [Selector Caching — Derived State](#8-selector-caching--derived-state)
9. [Normalization — Deduplication Cache](#9-normalization--deduplication-cache)
10. [Lazy Initialization](#10-lazy-initialization)
11. [Cache Warming and Prefetching](#11-cache-warming-and-prefetching)
12. [Memory Management](#12-memory-management)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. What Memory Caching Is

```
COMPUTATION (without caching):
  Input → [expensive computation] → Output
  Each call: pays full computation cost
  N calls with same input: N × cost

MEMORY CACHING (memoization):
  Input → [cache lookup] → Output (if cached)
  Input → [expensive computation] → Output → [store in cache]
  First call: pays full cost
  Subsequent calls with same input: cache lookup cost (microseconds)

BENEFIT:
  Trade RAM for CPU time
  Correct when: same input → same output (pure functions)

RISK:
  Cache grows unboundedly (memory leak)
  Stale data if input/output relationship changes
  Cache key design mistakes (over-matching or under-matching)
```

---

## 2. Memoization — Function Result Caching

### Basic Memoize

```typescript
// Simple memoize: caches all results forever
function memoize<T extends (...args: unknown[]) => unknown>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args); // serialize args as cache key

    if (cache.has(key)) {
      return cache.get(key) as ReturnType<T>;
    }

    const result = fn(...args) as ReturnType<T>;
    cache.set(key, result);
    return result;
  }) as T;
}

// Usage:
const expensiveCalc = memoize((n: number): number => {
  // Simulate expensive work
  let result = 0;
  for (let i = 0; i < n * 1000; i++) result += Math.sqrt(i);
  return result;
});

expensiveCalc(1000); // computes: ~5ms
expensiveCalc(1000); // cache hit: < 0.01ms
expensiveCalc(2000); // computes (new input): ~10ms
expensiveCalc(1000); // cache hit: < 0.01ms
```

### Memoize with Custom Key Function

```typescript
// Better memoize: configurable key function
function memoizeWithKey<TArgs extends unknown[], TResult>(
  fn: (...args: TArgs) => TResult,
  keyFn: (...args: TArgs) => string,
): (...args: TArgs) => TResult {
  const cache = new Map<string, TResult>();

  return (...args: TArgs): TResult => {
    const key = keyFn(...args);

    if (cache.has(key)) return cache.get(key)!;

    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// Usage: complex objects as arguments
const getUserDisplayName = memoizeWithKey(
  (user: User): string => `${user.firstName} ${user.lastName}`,
  (user: User): string => user.id, // use stable ID as key, not JSON.stringify
);

// Usage: function with multiple meaningful args
const calculatePrice = memoizeWithKey(
  (product: Product, currency: Currency, quantity: number): number => {
    return convertCurrency(product.basePrice * quantity, "USD", currency);
  },
  (p, c, q) => `${p.id}:${c}:${q}`, // composite key
);
```

### Single-Argument Memoize (Fastest)

```typescript
// For functions with a single primitive argument: no JSON.stringify needed
function memoize1<T extends (arg: unknown) => unknown>(fn: T): T {
  const cache = new Map<unknown, ReturnType<T>>();

  return ((arg: Parameters<T>[0]): ReturnType<T> => {
    if (cache.has(arg)) return cache.get(arg) as ReturnType<T>;
    const result = fn(arg) as ReturnType<T>;
    cache.set(arg, result);
    return result;
  }) as T;
}

// Useful for: processing arrays, parsing strings, formatting values
const parseDate = memoize1((dateStr: string): Date => new Date(dateStr));
const formatCurrency = memoize1((cents: number): string =>
  new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(
    cents / 100,
  ),
);
```

---

## 3. LRU Cache — Bounded Eviction

An LRU (Least Recently Used) cache has a maximum size. When full, the least recently accessed entry is evicted.

```typescript
class LRUCache<K, V> {
  readonly #capacity: number;
  readonly #map: Map<K, V>; // Map preserves insertion order

  constructor(capacity: number) {
    if (capacity < 1) throw new Error("LRU capacity must be >= 1");
    this.#capacity = capacity;
    this.#map = new Map();
  }

  get(key: K): V | undefined {
    if (!this.#map.has(key)) return undefined;

    // Move to end (most recently used)
    const value = this.#map.get(key)!;
    this.#map.delete(key);
    this.#map.set(key, value);
    return value;
  }

  set(key: K, value: V): this {
    if (this.#map.has(key)) {
      this.#map.delete(key); // remove to re-insert at end
    } else if (this.#map.size >= this.#capacity) {
      // Evict LRU: Map's first entry
      const lruKey = this.#map.keys().next().value;
      this.#map.delete(lruKey);
    }

    this.#map.set(key, value);
    return this;
  }

  has(key: K): boolean {
    return this.#map.has(key);
  }

  delete(key: K): boolean {
    return this.#map.delete(key);
  }

  clear(): void {
    this.#map.clear();
  }

  get size(): number {
    return this.#map.size;
  }
}

// Usage: bounded memoize that won't grow forever
function memoizeLRU<TArgs extends unknown[], TResult>(
  fn: (...args: TArgs) => TResult,
  capacity: number = 100,
  keyFn: (...args: TArgs) => string = (...args) => JSON.stringify(args),
): (...args: TArgs) => TResult {
  const cache = new LRUCache<string, TResult>(capacity);

  return (...args: TArgs): TResult => {
    const key = keyFn(...args);
    const hit = cache.get(key); // get() also refreshes recency
    if (hit !== undefined) return hit;

    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// Usage:
const getProductById = memoizeLRU(
  (products: Product[], id: string) => products.find((p) => p.id === id),
  200, // max 200 entries
  (_, id) => id, // key by id only (products array reference ignored)
);
```

### LRU Cache with OnEvict Callback

```typescript
class LRUCacheWithEvict<K, V> extends LRUCache<K, V> {
  readonly #onEvict: (key: K, value: V) => void;

  constructor(capacity: number, onEvict: (key: K, value: V) => void) {
    super(capacity);
    this.#onEvict = onEvict;
  }

  override set(key: K, value: V): this {
    if (!this.has(key) && this.size >= this["#capacity"]) {
      const lruKey = this["#map"].keys().next().value;
      const lruVal = this["#map"].get(lruKey)!;
      this.#onEvict(lruKey, lruVal);
    }
    return super.set(key, value);
  }
}

// Usage: cleanup resources on eviction
const imageCache = new LRUCacheWithEvict<string, ImageBitmap>(
  50,
  (_key, bitmap) => bitmap.close(), // release GPU memory
);
```

---

## 4. TTL Cache — Time-Based Expiration

A TTL (Time-To-Live) cache automatically expires entries after a configurable duration.

```typescript
interface CacheEntry<V> {
  value: V;
  expiresAt: number;
}

class TTLCache<K, V> {
  readonly #ttlMs: number;
  readonly #map: Map<K, CacheEntry<V>>;
  readonly #maxSize: number;

  constructor(ttlMs: number, maxSize: number = Infinity) {
    this.#ttlMs = ttlMs;
    this.#maxSize = maxSize;
    this.#map = new Map();
  }

  set(key: K, value: V, ttlMs = this.#ttlMs): this {
    // Enforce max size (FIFO eviction)
    if (!this.#map.has(key) && this.#map.size >= this.#maxSize) {
      const firstKey = this.#map.keys().next().value;
      this.#map.delete(firstKey);
    }

    this.#map.set(key, {
      value,
      expiresAt: Date.now() + ttlMs,
    });
    return this;
  }

  get(key: K): V | undefined {
    const entry = this.#map.get(key);
    if (!entry) return undefined;

    if (Date.now() > entry.expiresAt) {
      this.#map.delete(key); // lazy deletion
      return undefined;
    }

    return entry.value;
  }

  has(key: K): boolean {
    return this.get(key) !== undefined;
  }

  delete(key: K): boolean {
    return this.#map.delete(key);
  }

  // Actively purge expired entries (call periodically for memory hygiene)
  prune(): number {
    const now = Date.now();
    let pruned = 0;

    for (const [key, entry] of this.#map) {
      if (now > entry.expiresAt) {
        this.#map.delete(key);
        pruned++;
      }
    }

    return pruned;
  }

  get size(): number {
    return this.#map.size;
  }
}

// Usage:
const apiResponseCache = new TTLCache<string, ApiResponse>(
  60_000, // 1 minute TTL
  500, // max 500 entries
);

async function fetchWithCache(url: string) {
  const cached = apiResponseCache.get(url);
  if (cached) return cached;

  const data = await fetch(url).then((r) => r.json());
  apiResponseCache.set(url, data);
  return data;
}

// Periodic pruning (don't rely on lazy deletion alone for long-lived caches)
setInterval(() => apiResponseCache.prune(), 5 * 60_000); // every 5 minutes
```

---

## 5. LFU Cache — Frequency-Based Eviction

An LFU (Least Frequently Used) cache evicts the entry accessed least often. Better than LRU when hot items aren't necessarily recent.

```typescript
class LFUCache<K, V> {
  readonly #capacity: number;
  #minFreq = 0;
  readonly #keyMap: Map<K, { value: V; freq: number }> = new Map();
  readonly #freqMap: Map<number, Set<K>> = new Map();

  constructor(capacity: number) {
    this.#capacity = capacity;
  }

  get(key: K): V | undefined {
    const entry = this.#keyMap.get(key);
    if (!entry) return undefined;

    this.#incrementFreq(key, entry);
    return entry.value;
  }

  set(key: K, value: V): void {
    if (this.#capacity <= 0) return;

    if (this.#keyMap.has(key)) {
      const entry = this.#keyMap.get(key)!;
      entry.value = value;
      this.#incrementFreq(key, entry);
      return;
    }

    if (this.#keyMap.size >= this.#capacity) {
      // Evict least frequent
      const minFreqSet = this.#freqMap.get(this.#minFreq)!;
      const evictKey = minFreqSet.values().next().value;
      minFreqSet.delete(evictKey);
      this.#keyMap.delete(evictKey);
    }

    this.#keyMap.set(key, { value, freq: 1 });
    if (!this.#freqMap.has(1)) this.#freqMap.set(1, new Set());
    this.#freqMap.get(1)!.add(key);
    this.#minFreq = 1;
  }

  #incrementFreq(key: K, entry: { value: V; freq: number }) {
    const oldFreq = entry.freq;
    const oldSet = this.#freqMap.get(oldFreq)!;
    oldSet.delete(key);
    if (oldSet.size === 0 && oldFreq === this.#minFreq) this.#minFreq++;

    entry.freq++;
    const newFreq = entry.freq;
    if (!this.#freqMap.has(newFreq)) this.#freqMap.set(newFreq, new Set());
    this.#freqMap.get(newFreq)!.add(key);
  }
}

// LFU shines when: hot items are accessed many times over a long period
// LRU shines when: recent items are more likely to be needed again
// For UI data: LRU is usually sufficient
```

---

## 6. WeakMap Caching — GC-Friendly

WeakMap holds object keys weakly — when the key object has no other references, it can be garbage collected along with the cache entry.

```typescript
// Perfect for: caching computed properties of objects you don't own
// When the object is GC'd: cache entry is automatically removed
// No memory leak risk

const computedMetricsCache = new WeakMap<HTMLElement, ComputedMetrics>();

function getComputedMetrics(element: HTMLElement): ComputedMetrics {
  if (computedMetricsCache.has(element)) {
    return computedMetricsCache.get(element)!;
  }

  const metrics = expensiveComputeMetrics(element);
  computedMetricsCache.set(element, metrics);
  return metrics;
}
// When element is removed from DOM and GC'd: cache entry automatically removed

// Real-world: caching rendered output for DOM elements
const tooltipContentCache = new WeakMap<Element, string>();

function getTooltip(element: Element): string {
  if (tooltipContentCache.has(element)) {
    return tooltipContentCache.get(element)!;
  }

  const content = computeTooltipContent(element); // expensive
  tooltipContentCache.set(element, content);
  return content;
}
```

### WeakRef for Optional Caching

```typescript
// WeakRef: hold a reference to an object without preventing GC
// Use when you WANT the cached value to potentially be collected under memory pressure

class SoftCache<K, V extends object> {
  readonly #map = new Map<K, WeakRef<V>>();

  set(key: K, value: V): void {
    this.#map.set(key, new WeakRef(value));
  }

  get(key: K): V | undefined {
    const ref = this.#map.get(key);
    if (!ref) return undefined;

    const value = ref.deref(); // returns undefined if GC'd
    if (value === undefined) {
      this.#map.delete(key); // clean up dead entry
    }
    return value;
  }
}

// Use for: large objects that can be recomputed (images, parsed data)
// Browser can reclaim memory under pressure; code recomputes on next access
const processedImageCache = new SoftCache<string, ImageData>();
```

---

## 7. React useMemo and useCallback

React's built-in memoization hooks cache values and functions between renders.

### useMemo — Memoize Computed Values

```typescript
// Memoize expensive computations within a component
function ProductList({ products, searchQuery, sortField, sortDir }: Props) {
  // ✅ Recomputed only when dependencies change
  const filteredAndSorted = useMemo(() => {
    const filtered = products.filter(p =>
      p.name.toLowerCase().includes(searchQuery.toLowerCase())
    );
    return filtered.sort((a, b) => {
      const cmp = a[sortField] < b[sortField] ? -1 : a[sortField] > b[sortField] ? 1 : 0;
      return sortDir === 'desc' ? -cmp : cmp;
    });
  }, [products, searchQuery, sortField, sortDir]);

  return (
    <ul>
      {filteredAndSorted.map(p => <ProductItem key={p.id} product={p} />)}
    </ul>
  );
}
```

### When useMemo Is Worth Using

```typescript
// ✅ WORTH IT: expensive computation that runs on many renders
const expensiveResult = useMemo(() => {
  // > 1ms computation, triggered by frequently-changing parent
  return processLargeDataset(data); // 50ms on 10,000 items
}, [data]);

// ✅ WORTH IT: stable reference for React.memo or useEffect deps
const config = useMemo(
  () => ({
    theme: "dark",
    locale: userLocale,
  }),
  [userLocale],
);
// Without useMemo: new object every render → child always re-renders

// ❌ NOT WORTH IT: cheap computation
const sum = useMemo(() => a + b, [a, b]); // useMemo overhead > computation
// Just compute directly:
const sum = a + b;

// ❌ NOT WORTH IT: depends on everything (no real memoization)
const result = useMemo(() => transform(data), [data, user, settings, theme]);
// If all deps change on every render: never cached
```

### useCallback — Memoize Functions

```typescript
// ✅ Stable function reference for child components or effects
function ParentComponent({ userId }: { userId: string }) {
  const [data, setData] = useState<Data[]>([]);

  // ✅ Same function reference as long as userId doesn't change
  const handleDelete = useCallback(async (itemId: string) => {
    await deleteItem(userId, itemId);
    setData(prev => prev.filter(d => d.id !== itemId));
  }, [userId]); // only recreated when userId changes

  return <ItemList items={data} onDelete={handleDelete} />;
}

// useCallback is only useful when:
// 1. Passed to React.memo children (prevents their re-render)
// 2. Used in useEffect dependencies (prevents re-running the effect)
// 3. Referenced in complex hooks as dependency

// ❌ NOT WORTH IT: function only used inline, not passed as prop
const handleClick = useCallback(() => setOpen(o => !o), []); // no benefit
// Direct: () => setOpen(o => !o) is fine
```

---

## 8. Selector Caching — Derived State

Selectors compute derived state from store state. Caching prevents recomputation when inputs haven't changed.

### Reselect — Memoized Selectors

```typescript
import { createSelector } from "reselect";

// Base selectors (no computation)
const selectProducts = (state: RootState) => state.products.items;
const selectFilters = (state: RootState) => state.products.filters;
const selectSearchQuery = (state: RootState) => state.products.searchQuery;

// Composed memoized selector
const selectFilteredProducts = createSelector(
  [selectProducts, selectFilters, selectSearchQuery],
  (products, filters, query) => {
    // Only recomputed when products, filters, or query change
    return products
      .filter((p) => matchesFilters(p, filters))
      .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()));
  },
);

const selectProductCount = createSelector(
  [selectFilteredProducts],
  (products) => products.length, // recomputed only when filteredProducts changes
);

// Multiple instances: use createSelectorCreator or re-reselect for
// parameterized selectors (per-ID caches)
import { createSelectorCreator, LruMemoize } from "@reduxjs/toolkit";

const createLRUSelector = createSelectorCreator(LruMemoize, { maxSize: 100 });

const selectProductById = createLRUSelector(
  [selectProducts, (_state: RootState, id: string) => id],
  (products, id) => products.find((p) => p.id === id),
);
```

### Zustand Selector Optimization

```typescript
// Zustand: select only the specific slice you need
// Re-renders ONLY when that slice changes

// ❌ Subscribes to entire store state — re-renders on any change
const state = useStore(); // whole store
const count = state.cart.items.length;

// ✅ Subscribes only to cart item count
const itemCount = useStore((state) => state.cart.items.length);
// Component re-renders ONLY when item count changes

// ✅ Shallow comparison for derived objects
import { useShallow } from "zustand/react/shallow";

const { name, email } = useStore(
  useShallow((state) => ({ name: state.user.name, email: state.user.email })),
);
// Re-renders only when name or email changes (not on any other state change)
```

---

## 9. Normalization — Deduplication Cache

Normalization stores each entity once and references by ID — the ultimate deduplication cache.

```typescript
// BEFORE (denormalized): duplicate data, sync problems
const state = {
  posts: [
    { id: "1", title: "Hello", author: { id: "u1", name: "Alice" } },
    { id: "2", title: "World", author: { id: "u1", name: "Alice" } }, // duplicate!
    { id: "3", title: "Again", author: { id: "u2", name: "Bob" } },
  ],
};
// If Alice's name changes: must update every post that mentions her

// AFTER (normalized): entities stored once, referenced by ID
interface NormalizedState {
  users: Record<string, User>;
  posts: Record<string, Post>;
  postIds: string[];
}

const state: NormalizedState = {
  users: {
    u1: { id: "u1", name: "Alice" },
    u2: { id: "u2", name: "Bob" },
  },
  posts: {
    "1": { id: "1", title: "Hello", authorId: "u1" },
    "2": { id: "2", title: "World", authorId: "u1" },
    "3": { id: "3", title: "Again", authorId: "u2" },
  },
  postIds: ["1", "2", "3"],
};
// Alice's name change: update one entry in users
// Benefit: derived views (post + author) always consistent
```

### Normalization with Normalizr

```typescript
import { normalize, schema } from "normalizr";

const userSchema = new schema.Entity("users");
const postSchema = new schema.Entity("posts", { author: userSchema });
const postListSchema = new schema.Array(postSchema);

const apiResponse = [
  { id: "1", title: "Hello", author: { id: "u1", name: "Alice" } },
  { id: "2", title: "World", author: { id: "u1", name: "Alice" } },
];

const normalized = normalize(apiResponse, postListSchema);
// {
//   entities: {
//     users: { 'u1': { id: 'u1', name: 'Alice' } },
//     posts: { '1': { id: '1', title: 'Hello', author: 'u1' }, ... }
//   },
//   result: ['1', '2']
// }

// Selectors assemble the denormalized view when needed:
function selectPostWithAuthor(state: NormalizedState, postId: string) {
  const post = state.posts[postId];
  const author = state.users[post.authorId];
  return { ...post, author }; // assembled on read
}
```

---

## 10. Lazy Initialization

Lazy initialization computes and stores a value only on first access:

```typescript
// Pattern 1: Lazy singleton
class LazyExpensive {
  static #instance: LazyExpensive | null = null;

  static getInstance(): LazyExpensive {
    if (!this.#instance) {
      this.#instance = new LazyExpensive(); // computed once
    }
    return this.#instance;
  }

  private constructor() {
    // Expensive setup: parse config, build indices, etc.
  }
}

// Pattern 2: Lazy property
class SearchEngine {
  #data: SearchableData;
  #index: SearchIndex | null = null; // not built until first search

  get index(): SearchIndex {
    if (!this.#index) {
      this.#index = buildSearchIndex(this.#data); // built on first access
    }
    return this.#index;
  }

  search(query: string): Result[] {
    return this.index.search(query); // triggers lazy build if needed
  }

  // Invalidate when data changes
  updateData(newData: SearchableData) {
    this.#data = newData;
    this.#index = null; // reset — will rebuild on next search
  }
}

// Pattern 3: Lazy factory via closure
function lazyValue<T>(factory: () => T): () => T {
  let value: T | undefined;
  let computed = false;

  return (): T => {
    if (!computed) {
      value = factory();
      computed = true;
    }
    return value as T;
  };
}

const getConfig = lazyValue(() => parseHeavyConfig());
// Config parsed only when first called
```

---

## 11. Cache Warming and Prefetching

Warm the cache before the user needs it to eliminate perceived latency:

```typescript
// Prefetch on hover: data ready before navigation
function ProductCard({ productId }: { productId: string }) {
  const queryClient = useQueryClient();

  const prefetch = useCallback(() => {
    queryClient.prefetchQuery({
      queryKey: ['product', productId],
      queryFn:  () => productsApi.get(productId),
      staleTime: 30_000, // don't prefetch if recently fetched
    });
  }, [productId, queryClient]);

  return (
    <div onMouseEnter={prefetch}> {/* warm on hover */}
      {/* ... */}
    </div>
  );
}

// Prefetch next page during idle time
async function prefetchNextPage(currentPage: number, totalPages: number) {
  if (currentPage >= totalPages) return;

  const nextPage = currentPage + 1;

  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
      queryClient.prefetchQuery({
        queryKey: ['products', { page: nextPage }],
        queryFn:  () => productsApi.list({ page: nextPage }),
      });
    });
  }
}
```

---

## 12. Memory Management

### Monitor Cache Memory Usage

```javascript
// Approximate memory used by a cache
function estimateCacheMemory(cache) {
  let bytes = 0;
  for (const [key, value] of cache.entries()) {
    // Rough estimates
    bytes += key.length * 2; // string key: 2 bytes/char (UTF-16)
    bytes += JSON.stringify(value).length * 2; // value estimate
  }
  return bytes;
}

// Log cache statistics in development
function logCacheStats(name, cache) {
  if (process.env.NODE_ENV !== "development") return;

  console.group(`Cache: ${name}`);
  console.log("Entries:", cache.size);
  console.log("~Memory:", (estimateCacheMemory(cache) / 1024).toFixed(1), "KB");
  console.groupEnd();
}
```

### Cache Cleanup Strategies

```typescript
// Strategy 1: Bounded size (LRU eviction)
const imageCache = new LRUCache<string, ImageBitmap>(100);

// Strategy 2: Time-based expiration
const apiCache = new TTLCache<string, Response>(5 * 60_000); // 5 min TTL

// Strategy 3: Manual invalidation on data change
function invalidateProductCache(productId: string) {
  queryClient.invalidateQueries({ queryKey: ["product", productId] });
  localProductCache.delete(productId);
}

// Strategy 4: Module-level cleanup on navigation
router.on("beforeNavigation", () => {
  // Clear page-specific caches when leaving the page
  pageCache.clear();
});
```

---

## 13. Good Practices

### ✅ Always bound cache sizes

```typescript
// ✅ All caches have a maximum size
const imageCache = new LRUCache<string, string>(200); // max 200 images
const computedCache = new TTLCache<string, number>(60_000, 1000); // max 1000 entries
```

### ✅ Use structural keys for complex cache lookups

```typescript
// ✅ Consistent, deterministic key generation
function cacheKey(filters: SearchFilters): string {
  // Sort keys for consistency regardless of object property order
  const sorted = Object.keys(filters)
    .sort()
    .map((k) => `${k}=${filters[k as keyof SearchFilters]}`)
    .join("&");
  return `search:${sorted}`;
}
```

### ✅ Log cache misses in development to track hit rate

```typescript
function cachedFetch(url: string) {
  const cached = cache.get(url);
  if (cached) {
    if (process.env.NODE_ENV === "development")
      console.debug(`[Cache HIT] ${url}`);
    return cached;
  }
  if (process.env.NODE_ENV === "development")
    console.debug(`[Cache MISS] ${url}`);
  // ... fetch and store
}
```

---

## 14. Bad Practices

### ❌ Unbounded memoize — silent memory leak

```typescript
// ❌ Basic memoize: cache grows forever
const memoized = memoize(expensiveFunction);
// Called with 10,000 unique arguments: 10,000 entries in memory
// Never GC'd as long as the memoized function is referenced

// ✅ Always use LRU or TTL for long-lived memoized functions
const memoized = memoizeLRU(expensiveFunction, 500); // max 500 entries
```

### ❌ Using useMemo/useCallback everywhere "just in case"

```typescript
// ❌ useMemo on cheap computation: overhead > savings
const label = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
// String concatenation: ~0.001ms
// useMemo overhead: ~0.01-0.05ms
// Net: slower by 10-50×

// ✅ Only memoize when you've measured or when reference stability matters
const label = `${firstName} ${lastName}`; // direct
```

### ❌ Stale closure in cached functions

```typescript
// ❌ Cached function captures stale values via closure
const userId = "user-1";
const getItems = memoize(() => fetchUserItems(userId)); // userId captured at creation
// Later: userId changes to 'user-2', but cache returns 'user-1' results

// ✅ Pass values as arguments so they're part of the cache key
const getItems = memoize((uid: string) => fetchUserItems(uid));
getItems("user-1"); // cache key: 'user-1'
getItems("user-2"); // cache key: 'user-2' (separate entry)
```

---

## 15. Common Mistakes

### Mistake 1 — JSON.stringify as cache key for all types

```typescript
// ❌ JSON.stringify has edge cases:
JSON.stringify(undefined); // → undefined (not a string!)
JSON.stringify({ a: 1, b: 2 }); // → '{"a":1,"b":2}'
JSON.stringify({ b: 2, a: 1 }); // → '{"b":2,"a":1}' — different key, same data!
JSON.stringify(NaN); // → 'null'
JSON.stringify(Infinity); // → 'null'
JSON.stringify(new Date()); // → '"2024-01-15T..."' (stringified)
JSON.stringify(new Map([[1, 2]])); // → '{}' — loses Map data!

// ✅ Use stable serialization or custom key functions
import { stringify } from "fast-json-stable-stringify"; // sorts keys
const key = stringify(obj); // { a: 1, b: 2 } and { b: 2, a: 1 } → same key
```

### Mistake 2 — Caching mutable objects

```typescript
// ❌ Caching a reference to a mutable array
const cache = new Map();
const arr = [1, 2, 3];
cache.set("arr", arr);

arr.push(4); // mutates! cache.get('arr') now returns [1,2,3,4]
// Cache entry silently "updated" — no consistency guarantee

// ✅ Cache immutable copies
cache.set("arr", Object.freeze([...arr]));
// Or: cache.set('arr', JSON.parse(JSON.stringify(arr))); // deep clone
```

### Mistake 3 — Not clearing caches on user logout

```typescript
// ❌ User A's data remains in cache after logout
// User B logs in: might see cached data from User A

// ✅ Clear all user-specific caches on logout
async function logout() {
  await authService.logout();

  // Clear ALL user-specific caches
  userDataCache.clear();
  queryClient.clear(); // TanStack Query
  sessionStorage.clear();
}
```

---

## 16. Interview-Level Explanation

> **"What is memoization? How do you implement a bounded in-memory cache?"**

**Strong answer:**

> "Memoization is a caching technique for pure functions — functions where the same inputs always produce the same outputs. You cache the result the first time a function is called with a given set of arguments, and on subsequent calls with the same arguments, you return the cached result instead of recomputing. A simple memoize function stores results in a Map keyed by a serialized version of the arguments. The key design decision is how to serialize arguments into a cache key: `JSON.stringify` works for simple cases but has edge cases with undefined, NaN, and unsorted object keys.
>
> The problem with basic memoize is that the cache grows forever — it's an unbounded memory leak. For production use you always want a bounded cache. An LRU cache fixes this: when the cache is full and a new entry needs to be stored, the least recently used entry is evicted. In JavaScript, this is elegantly implemented using a Map — Maps iterate in insertion order, so the first key in the iteration is always the oldest. On a get() you delete and re-insert the entry to move it to the 'most recently used' end. On a set() when full, you delete the first key before inserting the new one.
>
> For time-sensitive data, a TTL cache expires entries after a configurable duration. The implementation stores an expiry timestamp alongside the value and checks it on get(). Expired entries can be lazily deleted on access or actively purged on an interval.
>
> In React, useMemo and useCallback are memoization primitives for the component render cycle. useMemo caches a computed value and only recomputes when dependencies change — valuable for expensive transformations of large datasets. useCallback caches a function reference — valuable when passing callbacks to memoized child components, because the same reference prevents unnecessary re-renders. The key discipline: only use these when you've measured the performance problem or when reference stability is genuinely important. Adding useMemo to a simple calculation adds more overhead than it saves.
>
> For global state, selector libraries like Reselect provide memoized selectors that recompute derived data only when relevant input state changes. This is the right layer for expensive data transformations in a Redux or Zustand store — compute once when state changes, serve many components the same cached result."

---

## 17. Exercises

### Exercise 1 — LRU vs LFU choice

For each scenario, decide whether LRU, LFU, or TTL caching is most appropriate:

```
a) Route component cache: cache the last 10 pages user visited
b) API response cache: data valid for 5 minutes, then stale
c) Product detail cache: some products are very popular, others rarely viewed
d) User preference cache: any setting can change at any time
e) Computed search results for the most common 50 queries
```

<details>
<summary>Answers</summary>

```
a) Route component cache → LRU (capacity=10)
   Recently visited pages are most likely to be revisited
   LRU evicts least recently visited — correct behavior

b) API response cache → TTL (ttlMs=5*60*1000)
   Data freshness is the primary concern — time-based expiration is correct
   After 5 minutes: stale regardless of how recently accessed

c) Product detail cache → LFU
   Popular products accessed many times → high frequency, should stay cached
   Rarely-viewed products: evicted when capacity is reached
   LRU would evict popular products if they weren't accessed "recently"
   LFU keeps the truly hot items cached

d) User preference cache → No cache (or no-cache with ETag)
   Preferences can change at any time and must be consistent
   Stale preferences would cause confusing behavior
   If caching: short TTL (30s) or no-cache with revalidation

e) Common search query cache → LFU with bounded size (capacity=50)
   50 most common queries stay cached
   New queries compete by frequency — rare ones evicted
   Could also use: LRU if recent queries are equally likely to repeat
```

</details>

---

## 🔗 Related Topics

- [`performance/07-memoization.md`](../performance/07-memoization.md) — Memoization for rendering performance
- [`caching/04-data-caching.md`](./04-data-caching.md) — Application-level data caching
- [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md) — Derived state with selectors
- [`javascript-core/08-memory-management.md`](../javascript-core/08-memory-management.md) — Memory management fundamentals

---

<div align="center">

**Next:** [`caching/04-data-caching.md`](./04-data-caching.md) →

</div>
