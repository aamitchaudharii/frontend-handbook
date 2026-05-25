# 02 — Virtualization & Windowing

> **"Rendering 50,000 rows in a table doesn't require 50,000 DOM nodes. It requires exactly as many DOM nodes as fit on the screen — plus a few either side. Everything else is math. That insight is the entire foundation of virtualization."**

Virtualization (also called windowing) is the technique of rendering only the visible portion of a large dataset, maintaining the illusion of a fully rendered list through scroll position management and DOM recycling. It's the difference between a list that renders in 16ms regardless of dataset size and one that brings the browser to its knees at 10,000 rows.

---

## 📚 Table of Contents

1. [The Problem — Why Large Lists Break UIs](#1-the-problem--why-large-lists-break-uis)
2. [The Core Idea — Only Render What's Visible](#2-the-core-idea--only-render-what-is-visible)
3. [Fixed-Height Row Virtualization — From Scratch](#3-fixed-height-row-virtualization--from-scratch)
4. [Variable-Height Row Virtualization](#4-variable-height-row-virtualization)
5. [The Overscan Technique](#5-the-overscan-technique)
6. [Scroll Position and Item Index Calculation](#6-scroll-position-and-item-index-calculation)
7. [DOM Recycling Within a Virtual List](#7-dom-recycling-within-a-virtual-list)
8. [Virtualized Grid](#8-virtualized-grid)
9. [Infinite Scroll with Virtualization](#9-infinite-scroll-with-virtualization)
10. [Measuring Virtualization Performance](#10-measuring-virtualization-performance)
11. [When Virtualization Is the Wrong Tool](#11-when-virtualization-is-the-wrong-tool)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Problem — Why Large Lists Break UIs

### The DOM Node Cost at Scale

Rendering 10,000 rows naively:

```javascript
// ❌ Naive approach: render everything
function renderAllItems(data) {
  const container = document.getElementById("list");
  const fragment = document.createDocumentFragment();

  data.forEach((item) => {
    const row = document.createElement("div");
    row.className = "row";
    row.innerHTML = `
      <span class="name">${item.name}</span>
      <span class="price">${item.price}</span>
      <button class="btn">View</button>
    `;
    fragment.appendChild(row);
  });

  container.appendChild(fragment);
}

renderAllItems(
  new Array(10_000).fill(null).map((_, i) => ({
    name: `Item ${i}`,
    price: `$${(Math.random() * 100).toFixed(2)}`,
  })),
);
```

Costs at 10,000 rows:

```
DOM nodes: 10,000 rows × ~4 nodes each = 40,000 DOM nodes
Memory:    40,000 × ~500 bytes = ~20MB just for DOM
Initial render: 200–800ms (parsing, layout, paint for all 40,000 nodes)
Scroll: smooth initially, degrades as browser manages all nodes
Style matching: O(40,000 × rules) per recalculation
Layout: O(40,000) minimum for any geometry change
```

The user can see perhaps 20–30 rows at once. The other 9,970+ rows are invisible, below the fold — but fully allocated in memory and considered by the browser for every style recalculation, layout, and paint.

### The Illusion of a Full List

Virtualization creates the illusion of rendering all items while only maintaining DOM nodes for visible items:

```
Visual result:                  Actual DOM:
┌─────────────────────┐         ┌─────────────────────┐
│ Row 0               │         │ Row 0               │ ← rendered
│ Row 1               │         │ Row 1               │ ← rendered
│ Row 2               │         │ Row 2               │ ← rendered
│ Row 3               │         │ Row 3               │ ← rendered
│ Row 4               │         │ Row 4               │ ← rendered
│ ...                 │         │ Row 5               │ ← rendered
│ [thousands of rows] │         │ Row 6               │ ← rendered
│ ...                 │         └─────────────────────┘
│ Row 9,997           │         Only 7 rows in DOM!
│ Row 9,998           │         But user sees full scroll height
│ Row 9,999           │         and can scroll to any row
└─────────────────────┘
```

---

## 2. The Core Idea — Only Render What's Visible

The foundation of virtualization is simple geometry:

```
Container height:  600px
Row height:        48px
Visible rows:      Math.ceil(600 / 48) = 13 rows

At scroll position scrollTop = 0:
  First visible row index: Math.floor(0 / 48) = 0
  Last visible row index:  Math.floor((0 + 600) / 48) = 12

At scroll position scrollTop = 1000:
  First visible row index: Math.floor(1000 / 48) = 20
  Last visible row index:  Math.floor((1000 + 600) / 48) = 33

At any scroll position:
  Render rows [startIndex ... endIndex]
  Only 13-14 rows regardless of total dataset size
```

### The Spacer Technique

To maintain the correct scroll height without rendering all items:

```
Total list height = totalItems × rowHeight
Total height for 10,000 items at 48px = 480,000px

Structure:
  <div class="virtual-list-container" style="height: 600px; overflow-y: scroll">
    <div class="virtual-list-spacer" style="height: 480000px; position: relative">
      <!-- Rendered rows positioned absolutely within spacer -->
      <div class="row" style="position: absolute; top: 960px">Row 20</div>
      <div class="row" style="position: absolute; top: 1008px">Row 21</div>
      ...
    </div>
  </div>

The spacer div has the full list height → scrollbar reflects correct position
The rendered rows are positioned absolutely within the spacer → appear at right position
```

---

## 3. Fixed-Height Row Virtualization — From Scratch

A complete, production-ready virtual list implementation for fixed-height rows:

```javascript
class VirtualList {
  constructor(options) {
    const {
      container, // DOM element to render into
      items, // array of data items
      itemHeight, // height of each row in px
      renderItem, // function(item, index) → DOM element
      overscan = 3, // extra rows to render above and below viewport
    } = options;

    this._container = container;
    this._items = items;
    this._itemHeight = itemHeight;
    this._renderItem = renderItem;
    this._overscan = overscan;

    this._scrollTop = 0;
    this._renderedRange = { start: 0, end: 0 };
    this._renderedNodes = new Map(); // index → DOM node
    this._pool = []; // recycled DOM nodes

    this._initDOM();
    this._bindEvents();
    this._render();
  }

  _initDOM() {
    const totalHeight = this._items.length * this._itemHeight;

    // Container: fixed height, scrollable
    this._container.style.cssText = `
      height: 100%;
      overflow-y: auto;
      position: relative;
      will-change: transform;
    `;

    // Spacer: full virtual height (scroll track)
    this._spacer = document.createElement("div");
    this._spacer.style.cssText = `
      height: ${totalHeight}px;
      position: relative;
      pointer-events: none;
    `;

    // Content container: absolute, holds rendered rows
    this._content = document.createElement("div");
    this._content.style.cssText = `
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
    `;

    this._spacer.appendChild(this._content);
    this._container.appendChild(this._spacer);
  }

  _bindEvents() {
    this._onScroll = () => {
      this._scrollTop = this._container.scrollTop;
      this._render();
    };

    this._container.addEventListener("scroll", this._onScroll, {
      passive: true,
    });
  }

  _getVisibleRange() {
    const { _scrollTop, _container, _itemHeight, _items, _overscan } = this;
    const containerHeight = _container.clientHeight;

    const startIndex = Math.max(
      0,
      Math.floor(_scrollTop / _itemHeight) - _overscan,
    );

    const endIndex = Math.min(
      _items.length - 1,
      Math.ceil((_scrollTop + containerHeight) / _itemHeight) + _overscan,
    );

    return { start: startIndex, end: endIndex };
  }

  _render() {
    const { start, end } = this._getVisibleRange();
    const { _renderedRange, _renderedNodes } = this;

    // Skip if range hasn't changed
    if (start === _renderedRange.start && end === _renderedRange.end) return;

    // Remove nodes no longer in range (recycle them)
    for (const [index, node] of _renderedNodes) {
      if (index < start || index > end) {
        node.remove();
        this._recycleNode(node);
        _renderedNodes.delete(index);
      }
    }

    // Add nodes for new range items
    for (let i = start; i <= end; i++) {
      if (!_renderedNodes.has(i)) {
        const node = this._getOrCreateNode(i);
        this._content.appendChild(node);
        _renderedNodes.set(i, node);
      }
    }

    // Update rendered range
    this._renderedRange = { start, end };
  }

  _getOrCreateNode(index) {
    const node =
      this._pool.length > 0 ? this._pool.pop() : document.createElement("div");

    // Position the row
    node.style.cssText = `
      position: absolute;
      top: ${index * this._itemHeight}px;
      left: 0;
      width: 100%;
      height: ${this._itemHeight}px;
      overflow: hidden;
    `;

    // Clear and populate with item content
    node.innerHTML = "";
    node.appendChild(this._renderItem(this._items[index], index));

    return node;
  }

  _recycleNode(node) {
    if (this._pool.length < 50) {
      // cap pool size
      this._pool.push(node);
    }
    // Otherwise: let GC collect it
  }

  // Public API
  scrollToIndex(index) {
    this._container.scrollTop = index * this._itemHeight;
  }

  updateItems(newItems) {
    this._items = newItems;
    this._spacer.style.height = `${newItems.length * this._itemHeight}px`;
    // Clear rendered nodes
    this._renderedNodes.forEach((node) => {
      node.remove();
      this._recycleNode(node);
    });
    this._renderedNodes.clear();
    this._renderedRange = { start: 0, end: 0 };
    this._render();
  }

  destroy() {
    this._container.removeEventListener("scroll", this._onScroll);
    this._container.innerHTML = "";
  }
}
```

### Usage Example

```javascript
const list = new VirtualList({
  container: document.getElementById("list-container"),
  items: new Array(100_000).fill(null).map((_, i) => ({
    id: i,
    name: `Item ${i}`,
    price: (Math.random() * 100).toFixed(2),
  })),
  itemHeight: 48,
  renderItem(item, index) {
    const row = document.createElement("div");
    row.className = "list-row";
    row.innerHTML = `
      <span class="row__index">${index}</span>
      <span class="row__name">${item.name}</span>
      <span class="row__price">$${item.price}</span>
    `;
    return row;
  },
  overscan: 5,
});

// Performance: same 15-20 DOM nodes regardless of whether you have
// 100 or 100,000 items in the dataset
```

---

## 4. Variable-Height Row Virtualization

Fixed-height virtualization is straightforward. Variable-height rows require measuring or estimating heights.

### Strategy 1 — Measure on First Render

```javascript
class VariableHeightVirtualList {
  constructor(options) {
    this._items = options.items;
    this._container = options.container;
    this._estimatedHeight = options.estimatedHeight ?? 48;
    this._renderItem = options.renderItem;
    this._overscan = options.overscan ?? 3;

    // Cache: index → measured height
    this._measuredHeights = new Map();
    // Cache: index → cumulative offset from top
    this._offsets = null; // computed lazily
    this._totalHeight = this._items.length * this._estimatedHeight;
  }

  _getOffset(index) {
    // Build offsets array if not cached
    if (!this._offsets) {
      this._offsets = new Float64Array(this._items.length + 1);
      this._offsets[0] = 0;
      for (let i = 0; i < this._items.length; i++) {
        const height = this._measuredHeights.get(i) ?? this._estimatedHeight;
        this._offsets[i + 1] = this._offsets[i] + height;
      }
      this._totalHeight = this._offsets[this._items.length];
    }
    return this._offsets[index];
  }

  _findIndexAtOffset(scrollTop) {
    // Binary search: find first item whose top edge >= scrollTop
    let low = 0,
      high = this._items.length;
    while (low < high) {
      const mid = Math.floor((low + high) / 2);
      if (this._getOffset(mid) < scrollTop) {
        low = mid + 1;
      } else {
        high = mid;
      }
    }
    return Math.max(0, low - 1);
  }

  _measureRenderedItems() {
    // After rendering, measure actual heights of visible items
    let needsRebuild = false;
    for (const [index, node] of this._renderedNodes) {
      const actualHeight = node.offsetHeight;
      const cachedHeight = this._measuredHeights.get(index);
      if (cachedHeight !== actualHeight) {
        this._measuredHeights.set(index, actualHeight);
        needsRebuild = true;
      }
    }
    if (needsRebuild) {
      this._offsets = null; // invalidate cache
      this._spacer.style.height = `${this._getOffset(this._items.length)}px`;
    }
  }
}
```

### Strategy 2 — ResizeObserver for Dynamic Heights

```javascript
// When items can change height (expandable rows, images loading):
const resizeObserver = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const index = parseInt(entry.target.dataset.virtualIndex, 10);
    const height = entry.contentRect.height;
    if (this._measuredHeights.get(index) !== height) {
      this._measuredHeights.set(index, height);
      this._offsets = null; // rebuild offset cache
      this._updateSpacerHeight();
    }
  }
});

// Observe each rendered item
renderedNode.dataset.virtualIndex = String(index);
resizeObserver.observe(renderedNode);

// Unobserve when recycled
resizeObserver.unobserve(recycledNode);
```

---

## 5. The Overscan Technique

Overscan renders additional rows above and below the visible area to prevent visible blank space during fast scrolling.

```
Viewport (600px tall):
  Visible rows: 12-13

Without overscan (overscan = 0):
  On fast scroll: new rows not rendered yet → blank area visible briefly
  Checkerboard/flash effect on fast scrolling

With overscan = 3:
  Render 3 extra rows above viewport top
  Render 3 extra rows below viewport bottom
  Total rendered: 12 + 3 + 3 = 18-19 rows
  Fast scrolling: pre-rendered rows already in DOM → smooth

Trade-off:
  More rows = smoother scroll but more DOM nodes and render time
  Optimal overscan: 3-5 rows for typical use cases
  High-speed scrolling: may need overscan = 5-10
```

```javascript
function getVisibleRange(
  scrollTop,
  containerHeight,
  itemHeight,
  itemCount,
  overscan = 3,
) {
  const visibleStart = Math.floor(scrollTop / itemHeight);
  const visibleEnd = Math.ceil((scrollTop + containerHeight) / itemHeight);

  return {
    start: Math.max(0, visibleStart - overscan),
    end: Math.min(itemCount - 1, visibleEnd + overscan),
  };
}
```

---

## 6. Scroll Position and Item Index Calculation

### Fixed-Height Calculations

```javascript
// Given scroll position → item index:
const startIndex = Math.floor(scrollTop / itemHeight);
// scrollTop=0, itemHeight=48: start = 0
// scrollTop=96, itemHeight=48: start = 2 (exactly at item 2)
// scrollTop=100, itemHeight=48: start = 2 (partway into item 2)

// Given item index → pixel position:
const itemTop = index * itemHeight;
// index=5, itemHeight=48: top = 240px

// Total scroll height:
const totalHeight = items.length * itemHeight;

// Items visible in container:
const visibleCount = Math.ceil(containerHeight / itemHeight);
```

### Scroll to a Specific Item

```javascript
scrollToIndex(index, alignment = 'start') {
  const itemTop = index * this._itemHeight;

  switch (alignment) {
    case 'start':
      // Item at the top of the viewport
      this._container.scrollTop = itemTop;
      break;

    case 'center':
      // Item in the center of the viewport
      this._container.scrollTop =
        itemTop - (this._container.clientHeight / 2) + (this._itemHeight / 2);
      break;

    case 'end':
      // Item at the bottom of the viewport
      this._container.scrollTop =
        itemTop - this._container.clientHeight + this._itemHeight;
      break;

    case 'auto':
      // Scroll only if item is not already visible
      const scrollTop = this._container.scrollTop;
      const viewEnd   = scrollTop + this._container.clientHeight;
      if (itemTop < scrollTop) {
        this._container.scrollTop = itemTop; // scroll up to show item
      } else if (itemTop + this._itemHeight > viewEnd) {
        this._container.scrollTop = itemTop + this._itemHeight - this._container.clientHeight;
      }
      break;
  }
}
```

---

## 7. DOM Recycling Within a Virtual List

As the user scrolls, rows leaving the viewport are detached and their DOM nodes are pooled for reuse by incoming rows.

```javascript
class NodePool {
  constructor(maxSize = 30) {
    this._pool   = [];
    this._maxSize = maxSize;
  }

  acquire() {
    return this._pool.pop() ?? null; // null = create fresh
  }

  release(node) {
    if (this._pool.length >= this._maxSize) return; // pool full → GC
    // Reset state before pooling
    node.className = '';
    node.innerHTML = '';
    node.removeAttribute('style');
    Object.assign(node.dataset, {}); // clear dataset
    this._pool.push(node);
  }

  get size() { return this._pool.length; }
}

// Integration in virtual list render cycle:
_render() {
  const range = this._getVisibleRange();

  // Phase 1: release nodes outside new range
  for (const [index, node] of this._renderedNodes) {
    if (index < range.start || index > range.end) {
      node.remove();
      this._pool.release(node);    // → back to pool
      this._renderedNodes.delete(index);
    }
  }

  // Phase 2: acquire nodes for new items
  for (let i = range.start; i <= range.end; i++) {
    if (!this._renderedNodes.has(i)) {
      const node = this._pool.acquire()        // from pool
                ?? document.createElement('div'); // or create fresh

      this._populateNode(node, this._items[i], i);
      this._content.appendChild(node);
      this._renderedNodes.set(i, node);
    }
  }
}
```

---

## 8. Virtualized Grid

Extending the same principle to 2D grids:

```javascript
class VirtualGrid {
  constructor({
    container,
    items,
    columns,
    itemWidth,
    itemHeight,
    renderItem,
  }) {
    this._container = container;
    this._items = items;
    this._columns = columns;
    this._itemWidth = itemWidth;
    this._itemHeight = itemHeight;
    this._renderItem = renderItem;

    const rows = Math.ceil(items.length / columns);
    this._totalHeight = rows * itemHeight;
    this._totalWidth = columns * itemWidth;

    this._init();
  }

  _getVisibleRows(scrollTop, containerHeight, overscan = 2) {
    const startRow = Math.max(
      0,
      Math.floor(scrollTop / this._itemHeight) - overscan,
    );
    const endRow = Math.min(
      Math.ceil(this._items.length / this._columns) - 1,
      Math.ceil((scrollTop + containerHeight) / this._itemHeight) + overscan,
    );
    return { startRow, endRow };
  }

  _render() {
    const { startRow, endRow } = this._getVisibleRows(
      this._container.scrollTop,
      this._container.clientHeight,
    );

    for (let row = startRow; row <= endRow; row++) {
      for (let col = 0; col < this._columns; col++) {
        const index = row * this._columns + col;
        if (index >= this._items.length) break;

        if (!this._rendered.has(index)) {
          const node = this._createNode(index, row, col);
          this._content.appendChild(node);
          this._rendered.set(index, node);
        }
      }
    }
  }

  _createNode(index, row, col) {
    const node = document.createElement("div");
    node.style.cssText = `
      position: absolute;
      top:    ${row * this._itemHeight}px;
      left:   ${col * this._itemWidth}px;
      width:  ${this._itemWidth}px;
      height: ${this._itemHeight}px;
    `;
    node.appendChild(this._renderItem(this._items[index], index));
    return node;
  }
}
```

---

## 9. Infinite Scroll with Virtualization

Combining virtualization with infinite scroll: load data as user approaches the end, without accumulating all rows in the DOM.

```javascript
class InfiniteVirtualList extends VirtualList {
  constructor(options) {
    super(options);
    this._loadMore = options.loadMore; // async fn() → new items
    this._isLoading = false;
    this._hasMore = true;
    this._threshold = options.threshold ?? 5; // load when N rows from end
  }

  _onScroll() {
    super._onScroll(); // update visible range

    if (this._hasMore && !this._isLoading) {
      this._checkLoadMore();
    }
  }

  async _checkLoadMore() {
    const lastVisible = this._renderedRange.end;
    const totalItems = this._items.length;

    if (lastVisible >= totalItems - this._threshold) {
      this._isLoading = true;
      this._showLoader();

      try {
        const newItems = await this._loadMore();

        if (newItems.length === 0) {
          this._hasMore = false;
        } else {
          this._items = [...this._items, ...newItems];
          // Update spacer height for new total
          this._spacer.style.height = `${this._items.length * this._itemHeight}px`;
          this._render();
        }
      } finally {
        this._isLoading = false;
        this._hideLoader();
      }
    }
  }
}
```

---

## 10. Measuring Virtualization Performance

### Node Count Verification

```javascript
// Verify virtualization is working correctly
function auditVirtualList(containerSelector) {
  const container = document.querySelector(containerSelector);
  const nodes = container.querySelectorAll("[data-virtual-item]");

  const totalItems = parseInt(container.dataset.totalItems, 10);
  const renderedCount = nodes.length;
  const efficiency = ((1 - renderedCount / totalItems) * 100).toFixed(1);

  console.log({
    totalItems,
    renderedInDOM: renderedCount,
    efficiency: `${efficiency}% of items NOT in DOM`,
    memoryEstimate: `~${(renderedCount * 0.5).toFixed(1)}KB for nodes`,
    naiveEstimate: `~${(totalItems * 0.5).toFixed(1)}KB would be naive`,
  });
}
```

### Scroll Performance

```javascript
// Measure scroll frame rate during virtual list scrolling
function measureScrollFPS(container, durationMs = 3000) {
  let frames = 0;
  let startTime = null;

  // Simulate programmatic scroll
  const maxScroll = container.scrollHeight - container.clientHeight;
  let scrollPos = 0;
  let direction = 1;

  const scrollInterval = setInterval(() => {
    scrollPos += 20 * direction;
    if (scrollPos >= maxScroll) direction = -1;
    if (scrollPos <= 0) direction = 1;
    container.scrollTop = scrollPos;
  }, 16);

  function countFrame(timestamp) {
    if (!startTime) startTime = timestamp;
    frames++;
    if (timestamp - startTime < durationMs) {
      requestAnimationFrame(countFrame);
    } else {
      clearInterval(scrollInterval);
      const fps = frames / (durationMs / 1000);
      console.log(`Scroll FPS: ${fps.toFixed(1)}`);
    }
  }

  requestAnimationFrame(countFrame);
}
```

---

## 11. When Virtualization Is the Wrong Tool

Virtualization adds complexity. Don't use it when:

### Small Lists (< 100 items)

```javascript
// ❌ Virtualization overhead for 50 items is not worth it
// 50 × 500 bytes = 25KB — negligible
// Virtual list code: 3KB+ and complexity
// Use pagination or plain rendering
```

### Heterogeneous Dynamic Height (Truly Unknown)

```javascript
// When items contain images, rich text, or complex layouts
// that cannot have heights estimated:
// → Virtualization requires measuring each item on render
// → For items that can change size (e.g., images loading):
//   complex ResizeObserver integration needed
// Alternative: pagination (simpler, more predictable)
```

### Accessibility Requirements

```javascript
// Screen readers depend on the full DOM tree for navigation
// Virtual lists can break:
//   - Page search (Ctrl+F): can't find non-rendered items
//   - Screen reader traversal: items not in DOM aren't announced
//   - Semantic HTML: list length mismatch

// Solutions:
// - ARIA: aria-rowcount="10000" aria-rowindex="1" on visible items
// - Screen reader announcements for loaded items
// - Test thoroughly with screen readers
```

### When Search/Filter Is Primary UX

```javascript
// User search: entire dataset must be queryable
// If search narrows to 20 items: no need for virtualization
// If search still returns 1000+ items: virtualization still needed
```

---

## 12. Good Practices

### ✅ Always overscan by 3–5 rows

```javascript
// ✅ Prevents blank flash during fast scroll
const range = getVisibleRange(scrollTop, containerHeight, itemHeight, count, 5);
```

### ✅ Use passive scroll listeners

```javascript
// ✅ Passive: scroll not blocked by event handler
container.addEventListener("scroll", onScroll, { passive: true });
```

### ✅ Throttle the render call

```javascript
// ✅ Don't re-render on every scroll pixel — batch with rAF
let rafId = null;
container.addEventListener(
  "scroll",
  () => {
    if (rafId) return;
    rafId = requestAnimationFrame(() => {
      rafId = null;
      render();
    });
  },
  { passive: true },
);
```

### ✅ Use `contain: strict` on list items

```css
/* ✅ Each item is fully isolated — no layout escape */
.virtual-list-item {
  contain: strict;
  height: 48px; /* known height */
}
```

### ✅ Implement `scrollToIndex` for navigation

```javascript
// ✅ Users expect to jump to specific items (search results, anchors)
list.scrollToIndex(targetIndex, "center");
```

---

## 13. Bad Practices

### ❌ Re-creating the entire list on each scroll event

```javascript
// ❌ Rebuilds ALL nodes on every scroll pixel
container.addEventListener("scroll", () => {
  container.innerHTML = ""; // destroy all nodes
  const fragment = document.createDocumentFragment();
  getVisibleItems().forEach((item) => {
    fragment.appendChild(createNode(item));
  });
  container.appendChild(fragment); // rebuild all nodes
});

// Correct approach: only add new nodes, only remove old nodes
```

### ❌ Using `position: relative` on many list items

```javascript
// ❌ relative positioning triggers layout for each item
.list-item {
  position: relative; // participates in normal flow
  height: 48px;
}
// With 10,000 items: browser checks all 10,000 for layout

// ✅ absolute positioning of items within a positioned container
// removes items from normal flow — no sibling layout interaction
.list-item {
  position: absolute; // positioned relative to container
  // top set by virtual list code
}
```

### ❌ Not recycling DOM nodes

```javascript
// ❌ Creating new elements on every render
for (let i = startIndex; i <= endIndex; i++) {
  const node = document.createElement("div"); // always creates new
  // ...
}

// ✅ Pool and reuse existing elements
const node = pool.acquire() ?? document.createElement("div");
```

### ❌ Reading offsetHeight inside the render loop

```javascript
// ❌ Forces layout per row during render
for (let i = startIndex; i <= endIndex; i++) {
  const node = createRow(items[i]);
  container.appendChild(node);
  const height = node.offsetHeight; // forced layout per row!
  cache.set(i, height);
}
```

---

## 14. Common Mistakes

### Mistake 1 — Incorrect total height calculation

```javascript
// ❌ Using integer division instead of multiplication
const totalHeight = items.length / itemHeight; // WRONG! should be ×
// spacer would be tiny (0.002px instead of 480000px)
// scrollbar would be useless

const totalHeight = items.length * itemHeight; // CORRECT
```

### Mistake 2 — Off-by-one in end index

```javascript
// ❌ Off-by-one: last visible row might be partially visible
const endIndex = Math.floor((scrollTop + containerHeight) / itemHeight);
// This gives floor — the item at this index might be cut off

// ✅ Use ceil for end index: ensures partially-visible rows are rendered
const endIndex = Math.ceil((scrollTop + containerHeight) / itemHeight);
```

### Mistake 3 — Not clamping indices to valid range

```javascript
// ❌ Negative or out-of-bounds index
const startIndex = Math.floor(scrollTop / itemHeight) - overscan;
// Can be negative if scrollTop is small and overscan is large

// ✅ Clamp to valid range
const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
const endIndex = Math.min(items.length - 1 /* ... */);
```

### Mistake 4 — Forgetting to handle scroll restoration

```javascript
// After data update, scroll position should be preserved
// or explicitly reset

updateItems(newItems) {
  const prevScrollTop = this._container.scrollTop;
  // ... update items ...
  // Restore scroll position (browser may reset it)
  this._container.scrollTop = prevScrollTop;
}
```

---

## 15. Interview-Level Explanation

> **"What is virtualization/windowing and how do you implement it?"**

**Strong answer:**

> "Virtualization is the technique of rendering only the DOM nodes for items currently visible in the viewport, rather than all items in a dataset. For a list of 100,000 rows where only 15 fit on screen, you maintain just 20–25 DOM nodes instead of 100,000 — regardless of dataset size.
>
> The core mechanism is straightforward geometry. You have a scrollable container with a fixed height, and inside it a spacer element whose height equals `items.length × rowHeight` — this gives you the correct scrollbar size without any content. Rendered rows are positioned absolutely within this spacer at `top: index × rowHeight`. On each scroll event, you calculate which indices are visible based on `scrollTop / rowHeight`, add rows that newly entered the viewport, and remove rows that left it.
>
> For smooth scrolling, you overscan by 3–5 rows in each direction — render slightly more than what's visible so fast scrolling doesn't show blank space. Scroll events should be passive (so they don't block browser scrolling) and batched with `requestAnimationFrame` to avoid redundant renders.
>
> DOM recycling is critical for performance: rather than creating new DOM nodes for each newly visible row, you pool the nodes of rows that scroll out of view, reset their content, and reuse them for incoming rows. This eliminates allocation cost and GC pressure entirely.
>
> For variable-height rows, you maintain a measurement cache. Items are rendered with an estimated height initially, then measured after first render and the cache is updated. A cumulative offset array (built as a prefix sum) enables O(log n) binary search to find which item is at any given scroll position.
>
> The technique matters when you have more than ~100-200 items. Below that, a simple pagination or full render is simpler and fine. Above a few thousand, virtualization is essentially mandatory for acceptable performance."

---

## 16. Exercises

### Exercise 1 — Build a minimal virtual list

Implement a virtual list that renders 10,000 items with fixed 40px height. Requirements:

- Only renders visible items + 3 overscan
- Correct scroll height maintained
- Uses event delegation for click handling
- Items show their index and a random color

```html
<div id="virtual-list" style="height: 500px; overflow-y: auto;"></div>
```

<details>
<summary>Solution</summary>

```javascript
function createVirtualList(container, count, itemHeight = 40) {
  const totalHeight = count * itemHeight;
  const colors = ["#e3f2fd", "#f3e5f5", "#e8f5e9", "#fff3e0", "#fce4ec"];

  // Spacer — full virtual height
  const spacer = document.createElement("div");
  spacer.style.cssText = `height: ${totalHeight}px; position: relative;`;
  container.appendChild(spacer);

  const renderedMap = new Map(); // index → node
  const pool = [];

  function getRange() {
    const scrollTop = container.scrollTop;
    const containerHeight = container.clientHeight;
    const start = Math.max(0, Math.floor(scrollTop / itemHeight) - 3);
    const end = Math.min(
      count - 1,
      Math.ceil((scrollTop + containerHeight) / itemHeight) + 3,
    );
    return { start, end };
  }

  function render() {
    const { start, end } = getRange();

    // Remove out-of-range nodes
    for (const [index, node] of renderedMap) {
      if (index < start || index > end) {
        node.remove();
        pool.push(node);
        renderedMap.delete(index);
      }
    }

    // Add new nodes
    for (let i = start; i <= end; i++) {
      if (!renderedMap.has(i)) {
        const node = pool.pop() ?? document.createElement("div");
        node.style.cssText = `
          position: absolute;
          top: ${i * itemHeight}px;
          left: 0; width: 100%;
          height: ${itemHeight}px;
          background: ${colors[i % colors.length]};
          display: flex; align-items: center;
          padding: 0 12px; box-sizing: border-box;
          border-bottom: 1px solid #ddd;
          cursor: pointer;
        `;
        node.dataset.index = i;
        node.textContent = `Item ${i.toLocaleString()}`;
        spacer.appendChild(node);
        renderedMap.set(i, node);
      }
    }
  }

  // Event delegation
  spacer.addEventListener("click", (e) => {
    const target = e.target.closest("[data-index]");
    if (target) alert(`Clicked item ${target.dataset.index}`);
  });

  // Scroll handler
  container.addEventListener(
    "scroll",
    () => {
      requestAnimationFrame(render);
    },
    { passive: true },
  );

  render();
}

createVirtualList(document.getElementById("virtual-list"), 10_000, 40);
```

</details>

---

### Exercise 2 — Benchmark: naive vs virtual

```javascript
// Run this benchmark in DevTools console
const container = document.createElement("div");
container.style.cssText = "height: 500px; overflow-y: auto; width: 400px;";
document.body.appendChild(container);

const DATA = Array.from({ length: 10_000 }, (_, i) => `Item ${i}`);

console.time("naive-render");
// Naive: render all 10,000 items
container.innerHTML = DATA.map(
  (d) =>
    `<div style="height:40px;padding:10px;border-bottom:1px solid #ddd">${d}</div>`,
).join("");
console.timeEnd("naive-render");

console.log("Naive DOM nodes:", container.querySelectorAll("div").length);
console.log(
  "Memory (estimate): ~",
  ((10_000 * 500) / 1024 / 1024).toFixed(1),
  "MB",
);
```

Compare with the virtual list from Exercise 1 — record:

- Initial render time
- DOM node count
- Memory in DevTools Memory tab
- Scroll frame rate (Performance tab)

---

## 🔗 Related Topics

- [`performance/01-dom-optimization.md`](./01-dom-optimization.md) — DOM fundamentals
- [`performance/12-large-data-rendering.md`](./12-large-data-rendering.md) — Large data rendering strategies
- [`javascript-core/09-garbage-collection.md`](../javascript-core/09-garbage-collection.md) — Object pooling for GC reduction
- [`performance/09-intersection-observer.md`](./09-intersection-observer.md) — IntersectionObserver for lazy loading
- [`projects/01-virtualized-table/`](../projects/01-virtualized-table/) — Full production virtualized table project

---

<div align="center">

**Next:** [`performance/03-layout-thrashing.md`](./03-layout-thrashing.md) →

</div>
