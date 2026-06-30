# 06 — Project: Infinite Scroll Image Gallery

> **"An infinite scroll gallery is deceptively simple in its happy path and deceptively hard in its edge cases: what happens when the user scrolls faster than images can load? What happens when they navigate away and back — do they lose their scroll position? What happens to memory after they've scrolled past 5,000 images? Each of these edge cases is where the real engineering happens."**

This project guide builds a Pinterest/Unsplash-style masonry image gallery: infinite scroll loading, a responsive masonry layout, lightbox viewing with keyboard navigation, image lazy-loading with blur-up placeholders, and scroll-position restoration — all while controlling memory growth for very long scroll sessions.

---

## 📚 What You'll Build

A gallery with: a responsive masonry grid layout, infinite scroll pagination, a full-screen lightbox with keyboard/swipe navigation, progressive image loading (blur-up placeholders), scroll position restoration on back-navigation, and DOM/memory management for sessions with thousands of loaded images.

---

## Requirements

```
FUNCTIONAL:
  - Masonry grid layout (variable aspect ratio images, no gaps)
  - Infinite scroll: load more images as the user approaches the bottom
  - Lightbox: click an image to view full-size, navigate with arrow keys/swipe
  - Blur-up placeholder while full image loads
  - Search/filter that resets the gallery with a new query

NON-FUNCTIONAL:
  - Scrolling 5,000+ images deep must not degrade scroll performance or
    exhaust memory
  - Navigating to an image's detail page and back must restore the exact
    scroll position (not reset to the top)
  - Masonry layout must not visibly "jump" as images load and their
    real dimensions become known
```

---

## Architecture Overview

```
COMPONENT TREE:
  <GalleryPage>
    <SearchBar />
    <MasonryGrid>
      <GalleryImage />            (repeated, virtualized for memory at scale)
    <InfiniteScrollSentinel />   (IntersectionObserver trigger)
    <Lightbox />                 (full-screen overlay, conditionally rendered)

KEY ARCHITECTURE DECISIONS:
  1. Masonry layout: CSS columns vs JS-computed absolute positioning
     — addressed in Step 1
  2. Memory management for long scroll sessions — addressed in Step 4
     (this is the most commonly overlooked requirement in gallery apps)
  3. Scroll restoration — addressed in Step 5
```

---

## Step 1 — Masonry Layout: CSS Columns vs JS Positioning

```css
/* OPTION 1: CSS columns (simple, but items flow top-to-bottom within
   each column, not left-to-right — order may look unintuitive) */
.masonry-css {
  column-count: 4;
  column-gap: 16px;
}
.masonry-css .item {
  break-inside: avoid;
  margin-bottom: 16px;
}
/* Pros: zero JS needed, works with any image dimensions automatically
   Cons: visual reading order is column-by-column, not row-by-row, which
   can feel unnatural; harder to control exact balance between columns */
```

```jsx
// OPTION 2: JS-computed absolute positioning (full control, more complex)
// Used by Pinterest-style "true masonry" layouts where reading order
// matters and column balance needs to be precise

function useMasonryLayout(items, columnCount, gap = 16) {
  const [positions, setPositions] = useState([]);
  const itemRefs = useRef(new Map());

  useLayoutEffect(() => {
    const columnHeights = new Array(columnCount).fill(0);
    const newPositions = items.map((item) => {
      // Place each item in the SHORTEST current column (balances heights)
      const shortestColumn = columnHeights.indexOf(Math.min(...columnHeights));
      const columnWidth =
        (containerWidth - gap * (columnCount - 1)) / columnCount;

      const x = shortestColumn * (columnWidth + gap);
      const y = columnHeights[shortestColumn];

      const itemHeight = (item.height / item.width) * columnWidth; // maintain aspect ratio
      columnHeights[shortestColumn] += itemHeight + gap;

      return { id: item.id, x, y, width: columnWidth, height: itemHeight };
    });

    setPositions(newPositions);
  }, [items, columnCount, gap]);

  const containerHeight = Math.max(...positions.map((p) => p.y + p.height), 0);

  return { positions, containerHeight };
}

function MasonryGrid({ items, columnCount }) {
  const { positions, containerHeight } = useMasonryLayout(items, columnCount);

  return (
    <div style={{ position: "relative", height: containerHeight }}>
      {items.map((item, i) => {
        const pos = positions[i];
        if (!pos) return null;
        return (
          <div
            key={item.id}
            style={{
              position: "absolute",
              transform: `translate(${pos.x}px, ${pos.y}px)`, // compositor-friendly
              width: pos.width,
              height: pos.height,
            }}
          >
            <GalleryImage item={item} />
          </div>
        );
      })}
    </div>
  );
}
```

**Key decision:** the JS-positioned approach uses `transform: translate()` for positioning rather than `top`/`left`, even though the values are computed once per layout pass (not animated) — this is a defensive choice that keeps the door open for smooth re-layout animations (e.g., when a search filter changes the image set) without needing to refactor positioning logic later. It also requires that you know each image's aspect ratio (`width`/`height`) BEFORE layout — which means your API needs to return image dimensions as metadata, not just URLs, so layout can be computed before any image has actually loaded.

---

## Step 2 — Infinite Scroll with Intersection Observer

```jsx
function useInfiniteScroll(fetchNextPage, hasNextPage, isFetching) {
  const sentinelRef = useRef(null);

  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasNextPage && !isFetching) {
          fetchNextPage();
        }
      },
      { rootMargin: "1000px" }, // start loading WAY before the user reaches the bottom
    );

    observer.observe(sentinel);
    return () => observer.disconnect();
  }, [fetchNextPage, hasNextPage, isFetching]);

  return sentinelRef;
}

function GalleryPage({ query }) {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfiniteQuery({
      queryKey: ["gallery", query],
      queryFn: ({ pageParam = 0 }) =>
        galleryApi.search(query, { page: pageParam, pageSize: 40 }),
      getNextPageParam: (lastPage) =>
        lastPage.hasMore ? lastPage.nextPage : undefined,
    });

  const sentinelRef = useInfiniteScroll(
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  );
  const allItems = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <>
      <MasonryGrid items={allItems} columnCount={4} />
      <div ref={sentinelRef} style={{ height: 1 }} />
      {isFetchingNextPage && <LoadingSpinner />}
    </>
  );
}
```

**Key decision:** `rootMargin: '1000px'` triggers the next page load when the sentinel is still 1000px BELOW the visible viewport, not when it actually scrolls into view — this is a generous prefetch margin that hides loading latency behind the user's scrolling, so by the time they actually reach the bottom of the loaded content, the next page has typically already arrived. The exact margin should be tuned based on typical scroll speed and API latency.

---

## Step 3 — Progressive Image Loading (Blur-Up)

```jsx
function GalleryImage({ item }) {
  const [isLoaded, setIsLoaded] = useState(false);

  return (
    <div className="gallery-image-container">
      {/* Tiny, heavily-compressed placeholder, shown immediately and blurred */}
      <img
        src={item.placeholderUrl} // e.g., a 20x20px base64-encoded thumbnail
        className="gallery-image-placeholder"
        style={{ filter: "blur(20px)", opacity: isLoaded ? 0 : 1 }}
        aria-hidden="true"
      />
      {/* Full-resolution image, fades in once loaded */}
      <img
        src={item.fullUrl}
        alt={item.alt}
        loading="lazy"
        onLoad={() => setIsLoaded(true)}
        className="gallery-image-full"
        style={{
          opacity: isLoaded ? 1 : 0,
          transition: "opacity 0.3s ease-out",
        }}
      />
    </div>
  );
}
```

```css
.gallery-image-container {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.gallery-image-placeholder,
.gallery-image-full {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: opacity 0.3s ease-out;
}
```

**Key decision:** the placeholder image is a TINY (often base64-inlined, so it requires zero additional network request) low-resolution version, blurred via CSS — this is the same "LQIP" (Low Quality Image Placeholder) technique used by Medium, Unsplash, and most modern image-heavy sites. It gives instant visual feedback about the image's general content and color palette while the full-resolution version loads, avoiding both a jarring blank space AND a generic loading spinner that conveys no information about the actual image.

---

## Step 4 — Memory Management for Long Scroll Sessions (The Overlooked Requirement)

```
THE PROBLEM:
  After scrolling through 5,000 images, the DOM contains 5,000 <img>
  elements, each holding a decoded bitmap in memory. Even with lazy
  loading (loading="lazy" only avoids the NETWORK request until needed),
  once an image HAS loaded, the browser keeps its decoded bitmap in
  memory for as long as the <img> element exists in the DOM. This is
  the single most common cause of long-session gallery apps consuming
  gigabytes of RAM and eventually crashing the tab.

SOLUTION: combine infinite scroll loading with virtualization that
ACTUALLY REMOVES off-screen images from the DOM (or at minimum, clears
their src), not just lazy-loads them in.
```

```jsx
function useImageVirtualization(items, viewportBuffer = 2000) {
  const [visibleRange, setVisibleRange] = useState({ start: 0, end: 40 });
  const containerRef = useRef(null);

  useEffect(() => {
    function updateVisibleRange() {
      const scrollTop = window.scrollY;
      const viewportHeight = window.innerHeight;

      // Find which items fall within [scrollTop - buffer, scrollTop + viewportHeight + buffer]
      // (requires item positions, computed by the masonry layout from Step 1)
      const start = findFirstVisibleIndex(items, scrollTop - viewportBuffer);
      const end = findLastVisibleIndex(
        items,
        scrollTop + viewportHeight + viewportBuffer,
      );

      setVisibleRange({ start, end });
    }

    window.addEventListener("scroll", throttle(updateVisibleRange, 200));
    updateVisibleRange();
    return () => window.removeEventListener("scroll", updateVisibleRange);
  }, [items, viewportBuffer]);

  return visibleRange;
}

function VirtualizedMasonryGrid({ items, columnCount }) {
  const { positions, containerHeight } = useMasonryLayout(items, columnCount);
  const { start, end } = useImageVirtualization(items);

  return (
    <div style={{ position: "relative", height: containerHeight }}>
      {items.slice(start, end).map((item, i) => {
        const pos = positions[start + i];
        if (!pos) return null;
        return (
          <div
            key={item.id}
            style={{
              position: "absolute",
              transform: `translate(${pos.x}px, ${pos.y}px)`,
              width: pos.width,
              height: pos.height,
            }}
          >
            <GalleryImage item={item} />
          </div>
        );
      })}
      {/* Items OUTSIDE [start, end] are not rendered AT ALL — their
          <img> elements (and decoded bitmaps) are released from memory */}
    </div>
  );
}
```

**Key decision:** notice that the masonry layout computation (`useMasonryLayout`) still runs against the FULL item list (so the overall container height and every item's POSITION is known), but the RENDERING (`items.slice(start, end)`) only materializes DOM nodes for the visible window. This is structurally the same pattern as the variable-height virtualized list from [`challenges/01-build-a-virtualized-list.md`](../challenges/01-build-a-virtualized-list.md), applied to a 2D masonry layout instead of a 1D list — the position calculation and the rendering decision are deliberately separated concerns.

---

## Step 5 — Lightbox with Keyboard Navigation

```jsx
function Lightbox({ items, currentIndex, onClose, onNavigate }) {
  useEffect(() => {
    function handleKeyDown(e) {
      if (e.key === "Escape") onClose();
      if (e.key === "ArrowRight")
        onNavigate(Math.min(currentIndex + 1, items.length - 1));
      if (e.key === "ArrowLeft") onNavigate(Math.max(currentIndex - 1, 0));
    }
    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [currentIndex, items.length, onClose, onNavigate]);

  // Preload adjacent images so navigation feels instant
  useEffect(() => {
    [currentIndex - 1, currentIndex + 1].forEach((i) => {
      if (items[i]) {
        const img = new Image();
        img.src = items[i].fullUrl;
      }
    });
  }, [currentIndex, items]);

  const currentItem = items[currentIndex];

  return createPortal(
    <div className="lightbox-overlay" onClick={onClose}>
      <button
        onClick={(e) => {
          e.stopPropagation();
          onNavigate(currentIndex - 1);
        }}
      >
        ‹
      </button>
      <img
        src={currentItem.fullUrl}
        alt={currentItem.alt}
        onClick={(e) => e.stopPropagation()}
      />
      <button
        onClick={(e) => {
          e.stopPropagation();
          onNavigate(currentIndex + 1);
        }}
      >
        ›
      </button>
      <button className="lightbox-close" onClick={onClose}>
        ×
      </button>
    </div>,
    document.body,
  );
}
```

**Key decision:** preloading the NEXT and PREVIOUS images (not just the current one) means that when the user presses the arrow key, the image is already in the browser's cache and displays instantly rather than showing a loading state — a small detail that makes lightbox browsing feel considerably more responsive, especially for users flipping through images quickly.

---

## Step 6 — Scroll Position Restoration

```jsx
// When navigating to an image's detail page and back, restore the
// exact scroll position (and ensure the SAME items are still loaded)
function useScrollRestoration(key) {
  const location = useLocation();
  const scrollPositions = useRef(new Map()); // module-level or context-level cache

  useEffect(() => {
    // Restore on mount, if we have a saved position for this key
    const savedPosition = scrollPositions.current.get(key);
    if (savedPosition !== undefined) {
      // Use requestAnimationFrame to wait for the DOM to actually be
      // populated with the same items before restoring scroll
      requestAnimationFrame(() => window.scrollTo(0, savedPosition));
    }

    // Save on unmount (navigating away)
    return () => {
      scrollPositions.current.set(key, window.scrollY);
    };
  }, [key]);
}

function GalleryPage({ query }) {
  useScrollRestoration(`gallery-${query}`);
  // ... rest of gallery implementation, including all previously-loaded
  // pages of items — restoring scroll position only works correctly if
  // the SAME items (not a freshly reset first page) are still present,
  // which is naturally true if using TanStack Query's caching (the
  // infinite query cache persists across remounts as long as the
  // queryKey matches and staleTime hasn't been exceeded)
}
```

**Key decision:** scroll restoration only works correctly because the underlying data (all the pages of items the user had loaded by scrolling) is ALSO cached — restoring a scroll position of "5000px down" is meaningless if the gallery resets back to just the first page of 40 items on remount. This is why this pattern depends on TanStack Query's cache persisting across the component unmount/remount cycle (default behavior, as long as you don't navigate away long enough for the cache to garbage-collect, controlled by `gcTime`).

---

## Performance and Memory Checklist

```
☐ Masonry layout computed from known image dimensions (not after-the-fact
  reflow once images load)
☐ Infinite scroll uses IntersectionObserver with generous rootMargin
☐ Images use blur-up placeholders for progressive loading
☐ Long scroll sessions virtualize the DOM (not just lazy-load) to control
  memory growth
☐ Lightbox preloads adjacent images for instant navigation
☐ Scroll position + loaded data both persist across navigation (not
  just scroll position alone)
```

---

## Extension Ideas

```
- Image upload with client-side compression before upload
- Collections/albums (group images, drag to reorder)
- Color-based search (find images with similar dominant colors)
- Download original / different resolution options
- Social features: likes, comments per image
- Masonry layout that re-flows smoothly (animated) when filters change
```

---

## 🔗 Related Topics

- [`challenges/01-build-a-virtualized-list.md`](../challenges/01-build-a-virtualized-list.md) — Core virtualization technique extended here to 2D
- [`performance/06-image-optimization.md`](../performance/06-image-optimization.md) — Image loading strategies
- [`anti-patterns/05-memory-leaks.md`](../anti-patterns/05-memory-leaks.md) — Memory management patterns

---

<div align="center">

**Next:** [`projects/07-multistep-form-wizard.md`](./07-multistep-form-wizard.md) →

</div>
