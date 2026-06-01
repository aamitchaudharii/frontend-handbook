# 12 — Large Data Rendering

> **"The browser was not designed to render a million rows. But with the right architecture — virtualization, progressive loading, chunked processing, and the right rendering technology — you can make it look like it was."**

Large data rendering is the convergence of every performance technique in this handbook. When you need to display, search, filter, and interact with 100,000+ records, you must combine virtualization, lazy loading, chunked processing, the right rendering primitive, and smart data structures. This document synthesizes those techniques into a complete architecture for large-data UIs.

---

## 📚 Table of Contents

1. [Defining "Large Data"](#1-defining-large-data)
2. [The Data-Rendering Stack Decision](#2-the-data-rendering-stack-decision)
3. [Virtual Scroll — The Non-Negotiable Base](#3-virtual-scroll--the-non-negotiable-base)
4. [Chunked Processing — Not Blocking the Main Thread](#4-chunked-processing--not-blocking-the-main-thread)
5. [Offloading to Web Workers](#5-offloading-to-web-workers)
6. [Progressive Data Loading](#6-progressive-data-loading)
7. [Efficient Filtering and Sorting](#7-efficient-filtering-and-sorting)
8. [Indexed Search](#8-indexed-search)
9. [Data Structures for Large Collections](#9-data-structures-for-large-collections)
10. [Real-Time Data — Streaming Updates](#10-real-time-data--streaming-updates)
11. [Large Tables — The Full Architecture](#11-large-tables--the-full-architecture)
12. [Large Graphs and Network Visualizations](#12-large-graphs-and-network-visualizations)
13. [Memory Management at Scale](#13-memory-management-at-scale)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. Defining "Large Data"

"Large" is relative to context. These thresholds are approximate guidance:

```
ROWS / ITEMS:
  < 100:      Any approach works. Use the simplest.
  100–500:    Direct DOM rendering. Maybe virtualize if rows are complex.
  500–5,000:  Virtualization recommended for smooth UX.
  5,000–50,000: Virtualization required. Chunked processing for operations.
  > 50,000:   Virtualization + worker offloading + progressive loading.
  > 1,000,000: Server-side pagination/filtering almost always required.

COLUMNS:
  < 20:       Normal table rendering.
  20–100:     Column virtualization + horizontal scroll.
  > 100:      Both row and column virtualization.

UPDATE FREQUENCY:
  < 1 update/second:  Direct DOM update.
  1–10 updates/second: Batched DOM updates.
  10–60 updates/second: requestAnimationFrame batching.
  > 60 updates/second: Canvas or WebGL required.
```

---

## 2. The Data-Rendering Stack Decision

```
Start here:
  What is the approximate record count?
  How often does data change?
  Is full-text search required?
  Are operations (sort, filter, group) client-side or server-side?

Decision tree:

Records < 1,000 AND changes < 1/second:
  → Direct DOM with virtualization (React, Vue, AG Grid)

Records 1,000–100,000 AND changes < 10/second:
  → Virtual DOM + Web Worker for operations
  → AG Grid, TanStack Virtual, react-window

Records > 100,000 OR changes > 10/second:
  → Canvas-based renderer (AG Grid with Canvas, Handsontable)
  → OR server-side pagination with local subset

Real-time streaming (thousands of updates/second):
  → Canvas or WebGL renderer
  → Ring buffers, time-series windowing

Full-text search:
  → Client-side index (Fuse.js for < 10,000, FlexSearch for > 10,000)
  → Server-side for > 100,000 records
```

---

## 3. Virtual Scroll — The Non-Negotiable Base

For any dataset over a few hundred items, render only what's visible. This is covered in depth in [`performance/02-virtualization-windowing.md`](./02-virtualization-windowing.md). The key principles for large data:

### Row Height Strategy

```javascript
// For 100,000 rows at 40px each:
// Total scroll height = 100,000 × 40 = 4,000,000px
// Many browsers cap at ~33,000,000px — fine here
// Chrome/Firefox actually allow much higher, but past ~10M px, precision issues

// Strategy for very tall virtual lists (> 10M px):
class ScaledVirtualList {
  #scaleFactor;
  #actualHeight;
  #scaledHeight;

  constructor(itemCount, itemHeight, maxScrollHeight = 10_000_000) {
    this.#actualHeight = itemCount * itemHeight;
    this.#scaledHeight = Math.min(this.#actualHeight, maxScrollHeight);
    this.#scaleFactor = this.#actualHeight / this.#scaledHeight;
  }

  // Convert scrollTop (scaled) to actual data offset
  getDataOffset(scrollTop) {
    return scrollTop * this.#scaleFactor;
  }

  // Convert actual data offset to scrollTop
  getScrollTop(dataOffset) {
    return dataOffset / this.#scaleFactor;
  }

  get spacerHeight() {
    return this.#scaledHeight;
  }
}
```

### Column Virtualization

For wide tables, virtualize columns too:

```javascript
class VirtualTable {
  constructor({ data, columns, rowHeight = 40, containerWidth }) {
    this._data = data;
    this._columns = columns;
    this._rowHeight = rowHeight;
    this._containerWidth = containerWidth;
    this._scrollLeft = 0;
    this._scrollTop = 0;
  }

  _getVisibleColumns() {
    let startX = 0;
    let start = -1;
    let end = -1;

    for (let i = 0; i < this._columns.length; i++) {
      const endX = startX + this._columns[i].width;

      if (end === -1 && endX >= this._scrollLeft) {
        start = Math.max(0, i - 1); // one column before visible (overscan)
      }
      if (endX > this._scrollLeft + this._containerWidth) {
        end = Math.min(this._columns.length - 1, i + 1); // one after
        break;
      }
      startX = endX;
    }

    return {
      start: start === -1 ? 0 : start,
      end: end === -1 ? this._columns.length - 1 : end,
    };
  }

  _getColumnOffset(colIndex) {
    return this._columns
      .slice(0, colIndex)
      .reduce((sum, col) => sum + col.width, 0);
  }
}
```

---

## 4. Chunked Processing — Not Blocking the Main Thread

Large data operations (sort, filter, transform) must never run synchronously on the main thread.

### Time-Sliced Processing

```javascript
/**
 * Process a large array in time-sliced chunks.
 * Yields to the event loop between chunks to keep UI responsive.
 *
 * @param items     - Array to process
 * @param processor - Function called for each item
 * @param chunkSize - Items per chunk (adjust based on item cost)
 * @param onProgress - Called with (processed, total) after each chunk
 */
async function processChunked(
  items,
  processor,
  { chunkSize = 500, onProgress = null, signal = null } = {},
) {
  const results = new Array(items.length);
  let processed = 0;

  while (processed < items.length) {
    if (signal?.aborted) throw new DOMException("Aborted", "AbortError");

    const end = Math.min(processed + chunkSize, items.length);

    // Process one chunk synchronously
    for (let i = processed; i < end; i++) {
      results[i] = processor(items[i], i);
    }

    processed = end;
    onProgress?.(processed, items.length);

    // Yield to event loop — allows rendering and input between chunks
    if (processed < items.length) {
      await new Promise((resolve) => setTimeout(resolve, 0));
      // Or: await scheduler.yield() (Scheduler API, Chrome 115+)
    }
  }

  return results;
}

// Usage: process 100,000 records without blocking
const results = await processChunked(
  rawData,
  (item) => computeExpensiveMetric(item),
  {
    chunkSize: 1000,
    onProgress: (done, total) => {
      progressBar.style.width = `${((done / total) * 100).toFixed(0)}%`;
    },
  },
);
```

### Budget-Based Chunking (Adaptive)

```javascript
// Adapt chunk size based on how long each chunk takes
async function processAdaptive(items, processor, targetMs = 8) {
  const results = new Array(items.length);
  let i = 0;
  let chunkSize = 100; // starting chunk size

  while (i < items.length) {
    const start = performance.now();
    const end = Math.min(i + chunkSize, items.length);

    for (let j = i; j < end; j++) {
      results[j] = processor(items[j]);
    }

    const elapsed = performance.now() - start;

    // Adjust chunk size to target ~8ms per chunk
    if (elapsed > 0) {
      chunkSize = Math.max(1, Math.floor(chunkSize * (targetMs / elapsed)));
    }

    i = end;
    if (i < items.length) await new Promise((r) => setTimeout(r, 0));
  }

  return results;
}
```

---

## 5. Offloading to Web Workers

For large data operations, the most robust approach is to move processing entirely off the main thread.

```javascript
// data-worker.js — runs in a Web Worker
let dataset = [];

self.onmessage = ({ data }) => {
  switch (data.type) {
    case "LOAD":
      dataset = data.payload;
      self.postMessage({ type: "LOADED", count: dataset.length });
      break;

    case "SORT": {
      const { field, direction } = data.payload;
      const sorted = [...dataset].sort((a, b) => {
        const cmp = a[field] < b[field] ? -1 : a[field] > b[field] ? 1 : 0;
        return direction === "desc" ? -cmp : cmp;
      });
      self.postMessage({ type: "SORTED", id: data.id, payload: sorted });
      break;
    }

    case "FILTER": {
      const { query, fields } = data.payload;
      const lower = query.toLowerCase();
      const filtered = dataset.filter((item) =>
        fields.some((field) =>
          String(item[field]).toLowerCase().includes(lower),
        ),
      );
      self.postMessage({
        type: "FILTERED",
        id: data.id,
        payload: filtered,
        query,
      });
      break;
    }

    case "AGGREGATE": {
      const { groupBy, metric } = data.payload;
      const groups = new Map();
      dataset.forEach((item) => {
        const key = item[groupBy];
        if (!groups.has(key)) groups.set(key, []);
        groups.get(key).push(item[metric]);
      });
      const result = Object.fromEntries(
        [...groups.entries()].map(([k, vals]) => [
          k,
          vals.reduce((a, b) => a + b, 0) / vals.length,
        ]),
      );
      self.postMessage({ type: "AGGREGATED", id: data.id, payload: result });
      break;
    }
  }
};
```

```javascript
// Main thread: proxy that routes to worker
class DataWorkerProxy {
  #worker = new Worker("./data-worker.js");
  #pending = new Map();
  #nextId = 0;

  constructor() {
    this.#worker.onmessage = ({ data }) => {
      const { id, type, payload } = data;
      if (id !== undefined) {
        const { resolve } = this.#pending.get(id) ?? {};
        this.#pending.delete(id);
        resolve?.(payload);
      }
    };
  }

  #call(type, payload) {
    const id = this.#nextId++;
    return new Promise((resolve) => {
      this.#pending.set(id, { resolve });
      this.#worker.postMessage({ type, id, payload });
    });
  }

  load(data) {
    this.#worker.postMessage({ type: "LOAD", payload: data });
  }

  sort(field, direction) {
    return this.#call("SORT", { field, direction });
  }
  filter(query, fields) {
    return this.#call("FILTER", { query, fields });
  }
  aggregate(groupBy, metric) {
    return this.#call("AGGREGATE", { groupBy, metric });
  }

  terminate() {
    this.#worker.terminate();
  }
}

// Usage
const worker = new DataWorkerProxy();
worker.load(hundredThousandRecords);

// Non-blocking — processing happens on worker thread
const filtered = await worker.filter("smith", ["firstName", "lastName"]);
renderVirtualList(filtered);
```

---

## 6. Progressive Data Loading

Don't fetch everything upfront. Load data as the user needs it.

### Cursor-Based Pagination

```javascript
class PaginatedDataSource {
  #cursor = null;
  #hasMore = true;
  #loading = false;
  #allLoaded = [];
  #pageSize;
  #fetchPage;

  constructor(fetchPage, pageSize = 100) {
    this.#fetchPage = fetchPage;
    this.#pageSize = pageSize;
  }

  async loadNext() {
    if (!this.#hasMore || this.#loading) return null;
    this.#loading = true;

    try {
      const page = await this.#fetchPage({
        cursor: this.#cursor,
        pageSize: this.#pageSize,
      });

      this.#allLoaded.push(...page.items);
      this.#cursor = page.nextCursor;
      this.#hasMore = page.hasMore;

      return page.items;
    } finally {
      this.#loading = false;
    }
  }

  get items() {
    return this.#allLoaded;
  }
  get hasMore() {
    return this.#hasMore;
  }
  get count() {
    return this.#allLoaded.length;
  }
}

// Integration with virtual scroll
const source = new PaginatedDataSource(fetchUsers, 200);
const virtualList = new VirtualList({
  items: source.items,
  itemHeight: 48,
  onNearBottom: async () => {
    const newItems = await source.loadNext();
    if (newItems) virtualList.updateItems(source.items);
  },
});
```

### Streaming Large Responses

```javascript
// Parse large JSON response as stream — don't wait for full response
async function* streamJsonArray(url) {
  const response = await fetch(url);
  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  let buffer = "";

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });

    // Extract complete JSON objects from buffer
    let start = buffer.indexOf("[") + 1;
    let i = start;

    while (i < buffer.length) {
      // Find complete JSON objects (simplified — real impl needs proper JSON parsing)
      if (buffer[i] === "{") {
        const end = buffer.indexOf("}", i) + 1;
        if (end > 0) {
          try {
            yield JSON.parse(buffer.slice(i, end));
          } catch {}
          i = end + 1; // skip comma
          buffer = buffer.slice(i);
          i = 0;
        } else break;
      } else {
        i++;
      }
    }
  }
}

// Render items as they stream in
let batch = [];
for await (const item of streamJsonArray("/api/large-dataset")) {
  batch.push(item);

  // Update UI every 500 items — don't thrash DOM
  if (batch.length >= 500) {
    appendToVirtualList(batch);
    batch = [];
  }
}
if (batch.length) appendToVirtualList(batch);
```

---

## 7. Efficient Filtering and Sorting

### Sorting with Comparison Caching

```javascript
// ❌ String comparison on every sort comparison — slow for large datasets
data.sort((a, b) => a.name.localeCompare(b.name));

// ✅ Pre-compute sort keys — O(n) preprocessing, O(n log n) sort comparisons
function sortWithCachedKeys(data, keyFn, direction = "asc") {
  // Step 1: compute sort keys for all items — O(n)
  const keyed = data.map((item, i) => ({ item, key: keyFn(item), i }));

  // Step 2: sort by pre-computed keys — faster comparison
  keyed.sort((a, b) => {
    const cmp = a.key < b.key ? -1 : a.key > b.key ? 1 : a.i - b.i;
    return direction === "asc" ? cmp : -cmp;
  });

  // Step 3: return sorted items
  return keyed.map((k) => k.item);
}

// Localecompare sort: pre-compute collation keys
const sorted = sortWithCachedKeys(
  users,
  (user) => user.name.toLocaleLowerCase(), // pre-computed
  "asc",
);
```

### Bitmap Filtering for Multiple Criteria

```javascript
// For repeated filtering with same criteria, use bitwise operations
// Each bit in a flag integer represents one filterable attribute

const STATUS = {
  ACTIVE: 0b001,
  INACTIVE: 0b010,
  PENDING: 0b100,
};

const CATEGORY = {
  ELECTRONICS: 0b001,
  CLOTHING: 0b010,
  FOOD: 0b100,
};

// Pre-compute flags per item — O(n) once
const items = rawData.map((item) => ({
  ...item,
  statusFlag: STATUS[item.status] ?? 0,
  categoryFlag: CATEGORY[item.category] ?? 0,
}));

// Filter with bitwise AND — O(n), no string comparisons
function filterItems(items, statusMask, categoryMask) {
  return items.filter(
    (item) =>
      (!statusMask || item.statusFlag & statusMask) &&
      (!categoryMask || item.categoryFlag & categoryMask),
  );
}

// "Show active or pending items in electronics or clothing"
const filtered = filterItems(
  items,
  STATUS.ACTIVE | STATUS.PENDING, // any of these statuses
  CATEGORY.ELECTRONICS | CATEGORY.CLOTHING, // any of these categories
);
```

---

## 8. Indexed Search

Full-text search over large datasets requires an index — linear scan is too slow.

### FlexSearch (Fast Client-Side Search)

```javascript
import FlexSearch from "flexsearch";

// Create index
const index = new FlexSearch.Document({
  document: {
    id: "id",
    index: ["name", "email", "company", "notes"],
    store: true, // store original documents
  },
  tokenize: "forward", // prefix search: "smi" finds "smith"
  resolution: 9, // scoring resolution
  cache: 100, // cache last 100 queries
});

// Index all records — do this in a worker for large datasets
async function buildIndex(records) {
  for (const record of records) {
    index.add(record);
  }
}

// Search
async function search(query, limit = 50) {
  const results = await index.searchAsync(query, {
    limit,
    enrich: true, // return stored documents
  });

  // Flatten results from multiple fields
  return results
    .flatMap((r) => r.result)
    .filter((item, i, arr) => arr.findIndex((x) => x.id === item.id) === i) // dedup
    .map((item) => item.doc);
}
```

### Trie for Autocomplete

```javascript
class TrieNode {
  constructor() {
    this.children = {};
    this.ids = new Set(); // item IDs with this prefix
    this.isEnd = false;
  }
}

class SearchTrie {
  #root = new TrieNode();

  insert(text, id) {
    const lower = text.toLowerCase();
    let node = this.#root;

    for (const char of lower) {
      if (!node.children[char]) {
        node.children[char] = new TrieNode();
      }
      node = node.children[char];
      node.ids.add(id); // every prefix node tracks matching IDs
    }

    node.isEnd = true;
  }

  search(prefix, limit = 20) {
    const lower = prefix.toLowerCase();
    let node = this.#root;

    for (const char of lower) {
      if (!node.children[char]) return [];
      node = node.children[char];
    }

    return [...node.ids].slice(0, limit);
  }
}

// Build trie (once, in worker)
const trie = new SearchTrie();
users.forEach((user) => {
  trie.insert(user.name, user.id);
  trie.insert(user.email, user.id);
  trie.insert(user.company, user.id);
});

// Search: O(prefix.length) lookup
const matchingIds = trie.search("smi"); // instant
const results = matchingIds.map((id) => userById.get(id));
```

---

## 9. Data Structures for Large Collections

### Map for O(1) Lookups

```javascript
// ❌ O(n) lookup in array
function getUserById(users, id) {
  return users.find((u) => u.id === id); // linear scan
}

// ✅ O(1) lookup with Map
const userMap = new Map(users.map((u) => [u.id, u]));
function getUserById(id) {
  return userMap.get(id); // hash lookup
}
```

### Flat Arrays vs Object Arrays

```javascript
// Object array (SoA → AoS conversion)
const items = [
  { x: 10, y: 20, value: 5.5 },
  { x: 30, y: 40, value: 7.2 },
  // ...100,000 more
];

// Structure of Arrays: better cache performance for bulk operations
const xs = new Float32Array(100_000);
const ys = new Float32Array(100_000);
const values = new Float64Array(100_000);

// Sorting by value: only accesses `values` array — better cache utilization
const indices = Float32Array.from({ length: 100_000 }, (_, i) => i);
indices.sort((a, b) => values[a] - values[b]);

// Result: sorted indices, can access xs[indices[0]] for first item
```

### Lazy Index Building

```javascript
// Don't build all indices upfront — build on first access
class LazyIndexedCollection {
  #data;
  #indices = new Map(); // fieldName → Map<value, Set<id>>

  constructor(data) {
    this.#data = data;
  }

  #buildIndex(field) {
    if (this.#indices.has(field)) return; // already built

    const index = new Map();
    this.#data.forEach((item) => {
      const key = item[field];
      if (!index.has(key)) index.set(key, new Set());
      index.get(key).add(item.id);
    });
    this.#indices.set(field, index);
  }

  filterBy(field, value) {
    this.#buildIndex(field); // built on first use
    const ids = this.#indices.get(field).get(value) ?? new Set();
    return [...ids].map((id) => this.getById(id));
  }

  getById(id) {
    return this.#data.find((item) => item.id === id);
  }
}
```

---

## 10. Real-Time Data — Streaming Updates

When data changes continuously, you need strategies to prevent the UI from being overwhelmed.

### Batched DOM Updates with rAF

```javascript
class RealtimeDataRenderer {
  #pending = new Map(); // id → latest data
  #rafId = null;
  #virtualList;

  constructor(virtualList) {
    this.#virtualList = virtualList;
  }

  // Called many times per second with individual updates
  update(id, data) {
    this.#pending.set(id, data); // overwrite: only latest matters

    if (!this.#rafId) {
      this.#rafId = requestAnimationFrame(() => this.#flush());
    }
  }

  #flush() {
    this.#rafId = null;

    if (this.#pending.size === 0) return;

    // Apply all pending updates at once (one DOM flush)
    for (const [id, data] of this.#pending) {
      this.#virtualList.updateItem(id, data);
    }

    this.#pending.clear();
  }
}
```

### Ring Buffer for Time-Series Data

```javascript
// Fixed-size circular buffer: only keep the last N points
class RingBuffer {
  #data;
  #head = 0;
  #size = 0;
  #capacity;

  constructor(capacity) {
    this.#capacity = capacity;
    this.#data = new Float64Array(capacity * 2); // [timestamp, value] pairs
  }

  push(timestamp, value) {
    const idx = (this.#head % this.#capacity) * 2;
    this.#data[idx] = timestamp;
    this.#data[idx + 1] = value;
    this.#head++;
    this.#size = Math.min(this.#size + 1, this.#capacity);
  }

  toArray() {
    const result = [];
    const start = this.#size < this.#capacity ? 0 : this.#head;
    for (let i = 0; i < this.#size; i++) {
      const idx = ((start + i) % this.#capacity) * 2;
      result.push({ time: this.#data[idx], value: this.#data[idx + 1] });
    }
    return result;
  }

  get length() {
    return this.#size;
  }
}

// Usage: keep last 1000 data points per metric
const buffer = new RingBuffer(1000);
websocket.on("metric", ({ timestamp, value }) => {
  buffer.push(timestamp, value);
  scheduleRender(buffer.toArray()); // debounced render
});
```

---

## 11. Large Tables — The Full Architecture

A production large-data table combines all the techniques:

```
Large Table Architecture (100,000 rows):

┌─────────────────────────────────────────────────────────────┐
│  DATA LAYER (Web Worker)                                     │
│  - Raw dataset in TypedArrays (Float32/Int32)               │
│  - Sort index (pre-computed or on-demand)                   │
│  - Filter bitmap (active filters as bitflags)               │
│  - Search index (FlexSearch or Trie)                        │
│  - Responds to: SORT, FILTER, SEARCH → returns sorted IDs   │
└─────────────────────────┬───────────────────────────────────┘
                          │ sorted/filtered ID array
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  VIRTUAL SCROLL ENGINE                                       │
│  - Maintains current sort/filter state                      │
│  - Computes visible row range from scroll position          │
│  - Only renders 20-30 DOM rows regardless of dataset size   │
│  - Recycles DOM rows as user scrolls                        │
└─────────────────────────┬───────────────────────────────────┘
                          │ visible row data
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  RENDERING LAYER                                             │
│  - DOM for < 50,000 rows (interactive, accessible)          │
│  - Canvas for > 50,000 rows or > 10 updates/second         │
│  - Column virtualization for > 30 columns                   │
└─────────────────────────────────────────────────────────────┘
```

### Column Freezing

```javascript
class FrozenColumnTable {
  constructor({ columns, frozenCount = 1 }) {
    this.frozenColumns = columns.slice(0, frozenCount);
    this.scrollColumns = columns.slice(frozenCount);

    // Two virtual scroll regions: frozen (no horizontal scroll) + scrolling
    this.frozenContainer = document.getElementById("frozen-cols");
    this.scrollContainer = document.getElementById("scroll-cols");
  }

  render(rows, scrollLeft) {
    // Frozen columns: always rendered, no horizontal scroll
    this.frozenColumns.forEach((col) => {
      renderColumn(this.frozenContainer, col, rows);
    });

    // Scrolling columns: virtualized horizontally
    const visibleCols = this._getVisibleColumns(scrollLeft);
    visibleCols.forEach((col) => {
      renderColumn(this.scrollContainer, col, rows);
    });
  }
}
```

---

## 12. Large Graphs and Network Visualizations

For network graphs (nodes + edges), DOM-based rendering fails at ~500 nodes.

### Rendering Strategy by Scale

```
< 100 nodes:    SVG — full interaction, animation, accessibility
100–1,000 nodes: SVG with virtualization (only render visible subgraph)
1,000–10,000:  Canvas 2D — fast, no DOM per node
> 10,000:      WebGL (Sigma.js, Graphology, Three.js)
```

### Canvas Graph Renderer

```javascript
class GraphRenderer {
  #canvas;
  #ctx;
  #nodes = [];
  #edges = [];
  #viewport = { x: 0, y: 0, scale: 1 };

  constructor(canvas) {
    this.#canvas = canvas;
    this.#ctx = canvas.getContext("2d");
  }

  setData(nodes, edges) {
    this.#nodes = nodes;
    this.#edges = edges;
  }

  render() {
    const { ctx, canvas } = { ctx: this.#ctx, canvas: this.#canvas };
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    const { x, y, scale } = this.#viewport;
    ctx.save();
    ctx.translate(x, y);
    ctx.scale(scale, scale);

    // Cull: only draw nodes/edges in viewport
    const vp = this.#getViewportBounds();

    // Draw edges first (below nodes)
    ctx.strokeStyle = "#ccc";
    ctx.lineWidth = 1 / scale;
    ctx.beginPath();
    for (const edge of this.#edges) {
      const s = this.#nodes[edge.source];
      const t = this.#nodes[edge.target];
      if (!this.#isEdgeVisible(s, t, vp)) continue;
      ctx.moveTo(s.x, s.y);
      ctx.lineTo(t.x, t.y);
    }
    ctx.stroke(); // ONE GPU call for all visible edges

    // Draw nodes
    ctx.fillStyle = "#4fc3f7";
    ctx.beginPath();
    for (const node of this.#nodes) {
      if (!this.#isNodeVisible(node, vp)) continue;
      ctx.moveTo(node.x + node.r, node.y);
      ctx.arc(node.x, node.y, node.r, 0, Math.PI * 2);
    }
    ctx.fill(); // ONE GPU call for all visible nodes

    ctx.restore();
  }

  #getViewportBounds() {
    const { x, y, scale } = this.#viewport;
    const { width, height } = this.#canvas;
    return {
      minX: -x / scale,
      minY: -y / scale,
      maxX: (-x + width) / scale,
      maxY: (-y + height) / scale,
    };
  }

  #isNodeVisible(node, vp) {
    return (
      node.x + node.r >= vp.minX &&
      node.x - node.r <= vp.maxX &&
      node.y + node.r >= vp.minY &&
      node.y - node.r <= vp.maxY
    );
  }

  #isEdgeVisible(s, t, vp) {
    // Simplified: check if either endpoint is in viewport
    return this.#isNodeVisible(s, vp) || this.#isNodeVisible(t, vp);
  }
}
```

---

## 13. Memory Management at Scale

### Snapshot-Based Undo

```javascript
// For large editable datasets, structural sharing instead of full copies
class VersionedDataStore {
  #versions = []; // [ { data: Map, timestamp } ]
  #current = new Map();
  #maxVersions = 50;

  set(id, value) {
    // Structural sharing: create new version but share unchanged data
    const newVersion = new Map(this.#current); // O(n) — can optimize with persistent data structures
    newVersion.set(id, value);

    this.#versions.push({ data: this.#current, timestamp: Date.now() });
    if (this.#versions.length > this.#maxVersions) {
      this.#versions.shift(); // evict oldest
    }

    this.#current = newVersion;
  }

  undo() {
    const prev = this.#versions.pop();
    if (prev) this.#current = prev.data;
  }

  get(id) {
    return this.#current.get(id);
  }
  get all() {
    return this.#current;
  }
}
```

### WeakRef for Optional Caches

```javascript
// Cache computed views without preventing GC of original data
class ComputedViewCache {
  #cache = new WeakMap(); // keyed by the data object itself

  getOrCompute(data, computeFn) {
    const cached = this.#cache.get(data);
    if (cached) {
      const val = cached.deref(); // may be undefined if GC'd
      if (val !== undefined) return val;
    }

    const computed = computeFn(data);
    this.#cache.set(data, new WeakRef(computed));
    return computed;
  }
}
```

---

## 14. Good Practices

### ✅ Server-side for truly large datasets

```javascript
// ✅ Don't fight the browser — move heavy lifting to the server
// For > 1,000,000 records: always paginate server-side
// Send only what fits on screen + a buffer

async function fetchPage({ page, pageSize, sort, filter }) {
  const params = new URLSearchParams({ page, pageSize, sort, filter });
  return fetch(`/api/users?${params}`).then((r) => r.json());
}
// Returns: { items: User[], total: number, hasMore: boolean }
```

### ✅ Measure actual thresholds in your app

```javascript
// Run performance tests with realistic data sizes
async function measureRenderTime(count) {
  const data = generateTestData(count);
  const start = performance.now();
  renderList(data);
  await new Promise((r) => requestAnimationFrame(r)); // wait for paint
  const elapsed = performance.now() - start;
  console.log(`${count} items: ${elapsed.toFixed(1)}ms`);
}

[100, 500, 1000, 5000, 10000].forEach(measureRenderTime);
// Identify exactly where your implementation starts to degrade
```

### ✅ Use appropriate data structure per operation

```javascript
// Read by ID: Map (O(1))
// Range queries: sorted array + binary search (O(log n))
// Full-text search: FlexSearch or Trie
// Set membership: Set (O(1))
// Sorted iteration: sorted array
```

---

## 15. Bad Practices

### ❌ Rendering entire dataset to check "how long it takes"

```javascript
// ❌ This will hang the browser for seconds
const table = document.getElementById("table");
allHundredThousandRecords.forEach((record) => {
  const row = document.createElement("tr");
  // ... append row
  table.appendChild(row);
});
// Then: "it's slow, how do I make it faster?"
// Answer: you shouldn't render all 100,000 rows in the DOM at once
```

### ❌ Sorting/filtering on every keystroke synchronously

```javascript
// ❌ Sorts 100,000 items on every keypress — blocks main thread
input.addEventListener("input", (e) => {
  const filtered = allData
    .filter((item) => item.name.includes(e.target.value))
    .sort((a, b) => a.name.localeCompare(b.name));
  renderList(filtered); // 100ms+ on every keystroke
});

// ✅ Debounce + worker
const debouncedSearch = debounce(async (query) => {
  const results = await worker.filter(query, ["name"]);
  renderList(results);
}, 150);

input.addEventListener("input", (e) => debouncedSearch(e.target.value));
```

### ❌ Storing duplicated data

```javascript
// ❌ Storing filtered and sorted copies in memory
const allData     = fetchAllData();        // 50MB
const filtered    = allData.filter(...);   // another 40MB
const sorted      = [...filtered].sort(); // another 40MB
// Total: 130MB for data that could be 50MB

// ✅ Store data once, compute views lazily via indices
const store    = allData;                 // 50MB
const filtered = filteredIndices;         // 400KB (index of matching IDs)
const sorted   = sortedIndices;           // 400KB (sorted order of IDs)
```

---

## 16. Common Mistakes

### Mistake 1 — Using `innerHTML` for large table updates

```javascript
// ❌ Rebuilds entire DOM on every update
table.innerHTML = rows.map((row) => `<tr>...</tr>`).join("");
// 100,000 rows × HTML string: slow parsing + full DOM reconstruction

// ✅ Targeted updates via virtual scroll — only update visible rows
virtualScroll.updateRows(changedIds);
```

### Mistake 2 — Ignoring the cost of serialization to workers

```javascript
// Sending 100,000 objects via postMessage: structured clone takes time
worker.postMessage(hundredThousandObjects); // serialize: ~100ms

// ✅ Transfer ArrayBuffers instead
const buffer = serializeToBuffer(data); // TypedArray packing
worker.postMessage({ buffer }, [buffer]); // zero-copy transfer
```

### Mistake 3 — Re-building search index on every filter change

```javascript
// ❌ Rebuilds entire search index on every user interaction
async function handleFilterChange(filters) {
  const data = await filterData(allRecords, filters);
  const index = buildSearchIndex(data); // expensive!
  return index;
}

// ✅ Build index once, filter the index
const index = buildSearchIndex(allRecords); // once
async function handleFilterChange(filters) {
  const filtered = applyFilters(allRecords, filters); // fast (bitmap)
  return filtered.map((item) => index.get(item.id)); // use existing index
}
```

---

## 17. Interview-Level Explanation

> **"How would you architect a frontend that needs to display and interact with 500,000 rows of data?"**

**Strong answer:**

> "The first question is whether all 500,000 rows need to be client-side at all. If operations like sorting, filtering, and search can be server-side, that's almost always the right answer — send the relevant page of data and metadata. But if the user legitimately needs to interact with the full dataset offline or with sub-100ms response time, here's the architecture.
>
> The rendering layer is virtual scroll — non-negotiable. You render only the 20-30 rows visible in the viewport, plus a small overscan buffer. The DOM has ~50 row nodes total regardless of whether the dataset has 500 or 500,000 rows. As the user scrolls, you recycle DOM nodes and repopulate them with new data.
>
> The data layer runs in a Web Worker. You load the full dataset into the worker and build indices: a sort index (pre-computed sorted order), a filter bitmap (bitflags per item for fast AND/OR filtering), and a search index (FlexSearch or a Trie for prefix matching). When the user sorts by a column, filters, or types a search query, the main thread sends a message to the worker. The worker responds with a sorted/filtered array of IDs. The main thread virtual scroll engine uses those IDs to look up the actual row data as needed.
>
> Data transfer between main thread and worker uses typed arrays where possible — Float32Array for numerical data, Int32Array for IDs. These can be transferred as Transferable objects with zero serialization cost.
>
> For real-time updates, I batch incoming changes in a Map (keyed by item ID, value is latest state) and flush to the virtual scroll on each rAF tick. This caps DOM mutations at 60 per second regardless of how fast data arrives.
>
> If update frequency exceeds what the DOM can handle (more than ~100 targeted updates per second), I'd switch the rendering layer from DOM to Canvas. The virtual scroll geometry stays the same, but rows become pixels in a canvas bitmap rather than DOM elements."

---

## 18. Exercises

### Exercise 1 — Design the architecture

You're building a real-time trading dashboard:

- 50,000 financial instruments in a sortable, filterable table
- Prices update for 500 instruments every second
- User needs: sort by any column, filter by exchange/type, search by name/ticker
- Must render < 20ms per update, never drop below 30fps

Describe the complete architecture including: rendering layer, data layer, update handling.

<details>
<summary>Solution</summary>

```
ARCHITECTURE:

DATA LAYER (Web Worker):
  - Load all 50,000 instruments into worker
  - Maintain TypedArray-backed sorted indices per column:
    Float64Array priceIndex (sorted order)
    Int32Array   tickerIndex (sorted by string comparison key)
  - Pre-compute filter bitmap per instrument (exchange flags, type flags)
  - Build FlexSearch index for name/ticker full-text search
  - On SORT/FILTER/SEARCH: return sorted/filtered ID array instantly (pre-indexed)

UPDATE HANDLING:
  - WebSocket delivers {id, price, change, volume} for 500 instruments/second
  - Batch incoming updates in a Map<id, latestData> on main thread
  - Flush to renderer on rAF tick (~60 times/second max)
  - For sorted views: mark "dirty" sort positions, don't resort on every update
  - Resort every 2-3 seconds or on user action (not per price tick)

RENDERING LAYER:
  - Virtual scroll: 20-30 DOM rows regardless of 50,000 total
  - DOM for interactive rows (expandable details, checkboxes, actions)
  - Canvas overlay for sparkline charts per row (too expensive as DOM)
  - Targeted row updates: only update DOM cells for visible rows in flush()
  - CSS transforms for price change flash animation (composite only)

COLUMN SORT OPTIMIZATION:
  - Pre-sorted indices: resort runs only when user changes sort column
  - "Live sort" for price column: ring buffer tracks sort position changes
    without full resort — update only rows whose rank changed significantly

RESULT:
  - 20ms for sort/filter (pre-indexed, worker)
  - <1ms DOM update per rAF flush (only 30 visible rows × 3 cells = 90 DOM writes)
  - Never drops below 30fps even under peak update load
```

</details>

---

## 🔗 Related Topics

- [`performance/02-virtualization-windowing.md`](./02-virtualization-windowing.md) — Virtual scroll implementation
- [`javascript-core/12-web-workers.md`](../javascript-core/12-web-workers.md) — Web Worker patterns
- [`performance/10-canvas-optimization.md`](./10-canvas-optimization.md) — Canvas rendering for large datasets
- [`performance/07-memoization.md`](./07-memoization.md) — Memoizing expensive computations
- [`performance/04-raf-optimization.md`](./04-raf-optimization.md) — rAF batching for real-time updates

---

<div align="center">

**`performance/` section complete!** 🎉

All 12 performance files are done.

**Next section:** [`system-design/`](../system-design/) →

</div>
