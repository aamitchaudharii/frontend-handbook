# 01 — Challenge: Build a Virtualized List from Scratch

> **"Challenges are different from exercises: there's no single function to implement, just a goal and a set of constraints that get harder as you go. The point isn't to finish quickly — it's to make the same design decisions a library author makes, including the ones that only reveal themselves once you hit the wall the first attempt didn't account for."**

This challenge builds a production-grade virtualized list component in stages, each adding a real-world requirement that breaks the previous stage's simplifying assumptions. By the end, you'll have built something approaching the sophistication of `react-window` or `@tanstack/virtual`, and understand exactly why those libraries make the design choices they do.

---

## The Goal

Build a `<VirtualList>` component that can render a scrollable list of 100,000 items while keeping the DOM node count under 50, supporting variable item heights, dynamic content, and smooth scroll performance — without external libraries.

---

## Stage 1 — Fixed-Height Virtualization (Foundation)

### Requirements

- Render a list of N items, each with a known, fixed height
- Only render items currently visible in the viewport (plus a small overscan buffer)
- Scrolling should feel native (use the browser's native scrollbar, not a custom one)

### Constraints

- No external virtualization libraries
- Must handle at least 100,000 items without degrading scroll performance
- DOM node count should stay roughly constant regardless of total item count

<details>
<summary>Hints</summary>

- The key insight: render a tall "spacer" element that has the FULL scrollable height, then absolutely position the visible items within it
- You need: total height, scroll position, visible range calculation, and a scroll listener
- Think about what happens when you compute `startIndex` from `scrollTop` — it's just division
</details>

<details>
<summary>Solution</summary>

```jsx
function VirtualList({ items, itemHeight, height, overscan = 3, renderItem }) {
  const [scrollTop, setScrollTop] = useState(0);
  const containerRef = useRef(null);

  const totalHeight = items.length * itemHeight;
  const visibleCount = Math.ceil(height / itemHeight);

  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const endIndex = Math.min(
    items.length,
    startIndex + visibleCount + overscan * 2,
  );

  const visibleItems = items.slice(startIndex, endIndex);

  function handleScroll(e) {
    setScrollTop(e.currentTarget.scrollTop);
  }

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      style={{ height, overflowY: "auto", position: "relative" }}
    >
      <div style={{ height: totalHeight, position: "relative" }}>
        {visibleItems.map((item, i) => {
          const index = startIndex + i;
          return (
            <div
              key={index}
              style={{
                position: "absolute",
                top: index * itemHeight,
                height: itemHeight,
                width: "100%",
              }}
            >
              {renderItem(item, index)}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**Why this works:** The outer container has a fixed height and `overflowY: auto`, which gives us native scrolling. The inner spacer div has the full theoretical height (`items.length * itemHeight`), which is what makes the scrollbar proportions correct — the browser thinks there's actually that much content. Inside it, we only place absolutely-positioned divs for the items currently in view, calculated directly from `scrollTop` via simple division.

**What breaks next:** This assumes every item is exactly the same height. Real lists rarely are.

</details>

---

## Stage 2 — Variable Height Items (Breaking the Fixed-Height Assumption)

### New Requirement

Items now have different, **unknown until rendered**, heights (think: a chat message list, or a feed with text and images of varying length).

### The Problem This Introduces

You can no longer compute `index * itemHeight` to find an item's position — you need a **running total of all previous item heights**, but you don't know any item's height until it's actually rendered (measured).

<details>
<summary>Hints</summary>

- You'll need a "measurement pass": render an item, measure it, store its height, then use that for positioning
- Consider a two-phase approach: estimate first (for instant initial render), then correct as real measurements come in
- A cache of measured heights, keyed by index, avoids re-measuring items you've already seen
- `ResizeObserver` or `getBoundingClientRect()` after render can give you actual heights
</details>

<details>
<summary>Solution</summary>

```jsx
function VariableVirtualList({
  items,
  estimatedHeight,
  height,
  overscan = 3,
  renderItem,
}) {
  const [scrollTop, setScrollTop] = useState(0);
  const measuredHeights = useRef(new Map()); // index → actual height
  const [, forceRender] = useReducer((x) => x + 1, 0); // trigger re-render after measurement

  // Compute the offset (top position) of any index, using measured heights
  // where available, estimated height as a fallback for unmeasured items
  function getItemOffset(index) {
    let offset = 0;
    for (let i = 0; i < index; i++) {
      offset += measuredHeights.current.get(i) ?? estimatedHeight;
    }
    return offset;
  }

  function getItemHeight(index) {
    return measuredHeights.current.get(index) ?? estimatedHeight;
  }

  function getTotalHeight() {
    let total = 0;
    for (let i = 0; i < items.length; i++) {
      total += getItemHeight(i);
    }
    return total;
  }

  // Binary search would be better here for large lists — see Stage 4
  function findStartIndex(scrollTop) {
    let offset = 0;
    for (let i = 0; i < items.length; i++) {
      const h = getItemHeight(i);
      if (offset + h > scrollTop) return Math.max(0, i - overscan);
      offset += h;
    }
    return items.length - 1;
  }

  const startIndex = findStartIndex(scrollTop);

  // Find endIndex by accumulating heights until we exceed the viewport
  let endIndex = startIndex;
  let accumulatedHeight = 0;
  while (
    endIndex < items.length &&
    accumulatedHeight < height + overscan * estimatedHeight
  ) {
    accumulatedHeight += getItemHeight(endIndex);
    endIndex++;
  }

  function ItemMeasurer({ index, children }) {
    const ref = useRef(null);

    useLayoutEffect(() => {
      if (!ref.current) return;
      const actualHeight = ref.current.getBoundingClientRect().height;
      const previouslyMeasured = measuredHeights.current.get(index);

      if (previouslyMeasured !== actualHeight) {
        measuredHeights.current.set(index, actualHeight);
        forceRender(); // re-render to apply corrected positions
      }
    }, [index]);

    return <div ref={ref}>{children}</div>;
  }

  return (
    <div
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{ height, overflowY: "auto", position: "relative" }}
    >
      <div style={{ height: getTotalHeight(), position: "relative" }}>
        {items.slice(startIndex, endIndex).map((item, i) => {
          const index = startIndex + i;
          return (
            <div
              key={index}
              style={{
                position: "absolute",
                top: getItemOffset(index),
                width: "100%",
              }}
            >
              <ItemMeasurer index={index}>
                {renderItem(item, index)}
              </ItemMeasurer>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**Why this works:** We render with an _estimated_ height initially, so the list renders instantly without waiting for measurements. After paint, `useLayoutEffect` measures the actual rendered height and corrects the cache. Because this runs in `useLayoutEffect` (before browser paint), corrections happen invisibly — the user never sees the wrong height flash on screen for items already measured in a previous render pass; only newly-scrolled-into-view items show a brief estimate-then-correct cycle.

**Performance problem introduced:** `getItemOffset` and `findStartIndex` are both O(n) — looping through every item before the target index. For 100,000 items, scrolling triggers this loop on every scroll event. This is the next thing to fix.

</details>

---

## Stage 3 — Performance: From O(n) to O(log n) Position Lookups

### The Problem

Stage 2's linear scan for offsets becomes the bottleneck at scale — every scroll event re-walks potentially tens of thousands of items.

### Requirement

Reduce position lookup from O(n) to O(log n) using a prefix-sum + binary search approach, while still supporting dynamically corrected heights.

<details>
<summary>Hints</summary>

- A prefix sum array lets you binary search for "which index does this scrollTop fall into?"
- You need to invalidate/recompute the prefix sum efficiently when a height correction comes in — but NOT recompute the entire array on every single correction if you can avoid it
- Consider: do you need exact prefix sums for ALL items, or can you maintain a cache that's "good enough" and only rebuild from the point of change forward?
</details>

<details>
<summary>Solution</summary>

```jsx
class HeightCache {
  #heights; // Map<index, height>
  #prefixSums; // array: prefixSums[i] = total height of items [0, i)
  #estimatedHeight;
  #dirtyFromIndex = Infinity; // prefix sums are valid up to (not including) this index

  constructor(itemCount, estimatedHeight) {
    this.#heights = new Map();
    this.#estimatedHeight = estimatedHeight;
    this.#prefixSums = new Array(itemCount + 1).fill(0);
    this.#rebuildFrom(0);
  }

  #rebuildFrom(startIndex) {
    // Recompute prefix sums starting from startIndex —
    // items before startIndex are assumed already correct
    let sum = this.#prefixSums[startIndex] ?? 0;
    for (let i = startIndex; i < this.#prefixSums.length - 1; i++) {
      sum += this.#heights.get(i) ?? this.#estimatedHeight;
      this.#prefixSums[i + 1] = sum;
    }
    this.#dirtyFromIndex = Infinity;
  }

  setHeight(index, height) {
    const current = this.#heights.get(index);
    if (current === height) return; // no change, skip rebuild
    this.#heights.set(index, height);
    // Mark dirty from this index forward — defer the actual rebuild
    this.#dirtyFromIndex = Math.min(this.#dirtyFromIndex, index);
  }

  // Call this before reading offsets, to apply any pending corrections
  flush() {
    if (this.#dirtyFromIndex !== Infinity) {
      this.#rebuildFrom(this.#dirtyFromIndex);
    }
  }

  getOffset(index) {
    this.flush();
    return this.#prefixSums[index];
  }

  getTotalHeight() {
    this.flush();
    return this.#prefixSums[this.#prefixSums.length - 1];
  }

  // Binary search: find the largest index whose offset <= scrollTop
  findIndexAtOffset(scrollTop) {
    this.flush();
    let lo = 0,
      hi = this.#prefixSums.length - 1;
    while (lo < hi) {
      const mid = (lo + hi + 1) >> 1;
      if (this.#prefixSums[mid] <= scrollTop) lo = mid;
      else hi = mid - 1;
    }
    return lo;
  }
}

function VirtualListV3({
  items,
  estimatedHeight,
  height,
  overscan = 3,
  renderItem,
}) {
  const cacheRef = useRef(null);
  if (!cacheRef.current) {
    cacheRef.current = new HeightCache(items.length, estimatedHeight);
  }
  const cache = cacheRef.current;

  const [scrollTop, setScrollTop] = useState(0);
  const [, forceRender] = useReducer((x) => x + 1, 0);

  const startIndex = Math.max(0, cache.findIndexAtOffset(scrollTop) - overscan);

  let endIndex = startIndex;
  let accumulated = 0;
  while (
    endIndex < items.length &&
    accumulated < height + overscan * estimatedHeight
  ) {
    accumulated += cache.getOffset(endIndex + 1) - cache.getOffset(endIndex);
    endIndex++;
  }

  function ItemMeasurer({ index, children }) {
    const ref = useRef(null);
    useLayoutEffect(() => {
      if (!ref.current) return;
      const actual = ref.current.getBoundingClientRect().height;
      const before = cache.getOffset(index);
      cache.setHeight(index, actual);
      // Only force a re-render if this correction actually changes
      // something visible (could be optimized further to check if
      // the correction affects currently-rendered items' positions)
      if (cache.getOffset(index) !== before) forceRender();
    }, [index]);
    return <div ref={ref}>{children}</div>;
  }

  return (
    <div
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{ height, overflowY: "auto", position: "relative" }}
    >
      <div style={{ height: cache.getTotalHeight(), position: "relative" }}>
        {items.slice(startIndex, endIndex).map((item, i) => {
          const index = startIndex + i;
          return (
            <div
              key={index}
              style={{
                position: "absolute",
                top: cache.getOffset(index),
                width: "100%",
              }}
            >
              <ItemMeasurer index={index}>
                {renderItem(item, index)}
              </ItemMeasurer>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**Why this works:** Binary search over a prefix-sum array gives O(log n) lookups for "which index is at this scroll position." The key design decision is **lazy invalidation**: when a height correction comes in, we don't immediately rebuild the entire prefix-sum array (which would be O(n) per correction — defeating the purpose). Instead, we mark the array dirty from the corrected index forward, and only rebuild when an offset is actually requested (`flush()`). In practice, this batches multiple corrections that happen in the same render pass into a single rebuild.

**Remaining limitation:** Every height correction still triggers an O(n) rebuild eventually (from the dirty point to the end). For a list where items are corrected one at a time as you scroll, in the worst case (corrections always at index 0), this degrades back to O(n) per scroll. A segment tree or Fenwick tree (Binary Indexed Tree) would give true O(log n) updates AND O(log n) queries — that's the production-grade data structure `@tanstack/virtual` and similar libraries use internally.

</details>

---

## Stage 4 — Smooth Scrolling and Overscan Tuning

### New Requirement

Fast scrolling (flicking) should not show blank gaps where items haven't rendered yet. Add intelligent overscan that scales with scroll velocity.

<details>
<summary>Hints</summary>

- Track scroll velocity: difference in scrollTop between two recent scroll events, divided by time elapsed
- When velocity is high: increase overscan to pre-render more buffer
- When velocity drops to near zero (user stopped scrolling): shrink overscan back down to save DOM nodes
</details>

<details>
<summary>Solution</summary>

```jsx
function useScrollVelocity() {
  const [velocity, setVelocity] = useState(0);
  const lastScrollTop = useRef(0);
  const lastTime = useRef(performance.now());

  function handleScroll(e) {
    const now = performance.now();
    const scrollTop = e.currentTarget.scrollTop;
    const dt = now - lastTime.current;
    const dy = scrollTop - lastScrollTop.current;

    if (dt > 0) {
      setVelocity(Math.abs(dy / dt)); // px per ms
    }

    lastScrollTop.current = scrollTop;
    lastTime.current = now;

    return scrollTop;
  }

  return { velocity, handleScroll };
}

function VirtualListV4({ items, estimatedHeight, height, renderItem }) {
  const [scrollTop, setScrollTop] = useState(0);
  const { velocity, handleScroll } = useScrollVelocity();

  // Dynamic overscan: more buffer when scrolling fast
  // velocity is px/ms; at 2 px/ms (fast flick), overscan grows significantly
  const dynamicOverscan = Math.min(20, Math.max(3, Math.round(velocity * 5)));

  function onScroll(e) {
    setScrollTop(handleScroll(e));
  }

  // ... rest uses dynamicOverscan instead of a fixed overscan value

  return (
    <div onScroll={onScroll} style={{ height, overflowY: "auto" }}>
      {/* ... render with dynamicOverscan ... */}
    </div>
  );
}
```

**Why this works:** During fast scrolling (flicking), the visible range changes rapidly between renders. A fixed small overscan (e.g., 3 items) isn't enough buffer — by the time React renders the new visible range, the user may have already scrolled past it, causing visible blank flashes. Scaling overscan with measured velocity means we pre-render more buffer exactly when it's needed (fast scroll) and shrink back to a minimal DOM footprint when scrolling is slow or stopped — a direct tradeoff between "extra DOM nodes" and "visible gaps," tuned dynamically instead of picking one fixed value that's wrong for either fast or slow scrolling.

</details>

---

## Stage 5 — Stretch Goals

```
ADDITIONAL CHALLENGES TO EXTEND THIS FURTHER:

1. HORIZONTAL VIRTUALIZATION
   Support virtualizing a wide row of items (horizontal scroll) using
   the same prefix-sum technique, swapped to the x-axis.

2. GRID VIRTUALIZATION (both axes)
   Virtualize both rows AND columns simultaneously for a spreadsheet-like
   grid with 100,000 rows × 50 columns. Only render the visible rectangle.

3. STICKY HEADERS WITHIN A VIRTUALIZED LIST
   Support section headers (e.g., grouped contacts list: "A", "B", "C")
   that stick to the top while their section is in view, accounting for
   virtualization (the header for an off-screen section shouldn't render
   unless it's the "currently sticky" one).

4. SCROLL-TO-INDEX
   Implement a scrollToIndex(index, align) function that programmatically
   scrolls the list so a specific item is visible — accounting for the
   fact that you may not know that item's exact offset yet if it hasn't
   been measured (requires an iterative "scroll, measure, correct, scroll
   again" approach for unmeasured variable-height items).

5. WINDOW-LEVEL VIRTUALIZATION
   Instead of a fixed-height scrollable container, virtualize against
   the BROWSER WINDOW's scroll position — useful for a long blog feed
   that should use native page scrolling rather than a nested scrollable div.
```

---

## What You Should Have Learned

```
1. Virtualization is fundamentally a "lie to the browser" — you create a
   spacer element with the FULL theoretical size so native scrollbars work
   correctly, while only actually rendering a small visible window.

2. Variable-height virtualization requires an estimate-then-correct cycle:
   you cannot know real heights without rendering, so you render with a
   guess, measure with useLayoutEffect (before paint, to avoid visible
   flicker), and correct.

3. Naive position lookups are O(n) per scroll event — at scale this becomes
   the bottleneck. Prefix sums + binary search get you to O(log n) lookups,
   though updates still require careful invalidation strategy (lazy
   rebuilding, or a proper Fenwick tree for true O(log n) updates).

4. Real-world virtualization needs to handle scroll velocity — fixed
   overscan is either wasteful (always large) or insufficient (always small)
   depending on scroll speed; dynamic overscan based on measured velocity
   solves both cases.

5. This is exactly the kind of problem where understanding the underlying
   data structure (prefix sums, binary search, Fenwick trees) directly
   translates to measurably different runtime performance — not a
   theoretical concern but a real difference between smooth and janky
   scrolling at scale.
```

---

## 🔗 Related Topics

- [`performance/12-large-data-rendering.md`](../performance/12-large-data-rendering.md)
- [`exercises/02-react-exercises.md`](../exercises/02-react-exercises.md) — Exercise 3.2 (simpler version of this challenge)
- [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md)

---

<div align="center">

**Next:** [`challenges/02-build-a-state-management-library.md`](./02-build-a-state-management-library.md) →

</div>
