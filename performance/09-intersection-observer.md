# 09 — IntersectionObserver

> **"Before IntersectionObserver, detecting element visibility meant scroll event listeners, getBoundingClientRect in loops, and forced synchronous layouts on every scroll. IntersectionObserver replaces all of that with a single asynchronous callback that fires when you actually need it."**

IntersectionObserver is one of the most impactful performance APIs in the browser. It asynchronously reports when an element enters or exits a viewport or container, without forcing layout recalculation and without blocking the main thread. This document covers the full API, every practical use case, and the patterns that make IntersectionObserver-powered features feel effortless.

---

## 📚 Table of Contents

1. [The Problem It Solves](#1-the-problem-it-solves)
2. [The IntersectionObserver API](#2-the-intersectionobserver-api)
3. [IntersectionObserverEntry — Understanding the Data](#3-intersectionobserverentry--understanding-the-data)
4. [threshold — Controlling When Callbacks Fire](#4-threshold--controlling-when-callbacks-fire)
5. [rootMargin — Expanding and Contracting the Root](#5-rootmargin--expanding-and-contracting-the-root)
6. [Lazy Image Loading](#6-lazy-image-loading)
7. [Infinite Scroll](#7-infinite-scroll)
8. [Sticky Header Detection](#8-sticky-header-detection)
9. [Animation on Scroll](#9-animation-on-scroll)
10. [Read Progress Indicator](#10-read-progress-indicator)
11. [Pausing Off-Screen Animations and Videos](#11-pausing-off-screen-animations-and-videos)
12. [Ad Impression Tracking](#12-ad-impression-tracking)
13. [One Observer vs Many](#13-one-observer-vs-many)
14. [Combining with ResizeObserver](#14-combining-with-resizeobserver)
15. [Good Practices](#15-good-practices)
16. [Bad Practices](#16-bad-practices)
17. [Common Mistakes](#17-common-mistakes)
18. [Interview-Level Explanation](#18-interview-level-explanation)
19. [Exercises](#19-exercises)

---

## 1. The Problem It Solves

### The Old Way — Scroll Events

Before IntersectionObserver, visibility detection looked like this:

```javascript
// ❌ Classic scroll-based visibility detection
function isVisible(element) {
  const rect = element.getBoundingClientRect(); // forces layout
  return (
    rect.top >= 0 &&
    rect.left >= 0 &&
    rect.bottom <= window.innerHeight &&
    rect.right <= window.innerWidth
  );
}

window.addEventListener("scroll", () => {
  images.forEach((img) => {
    if (isVisible(img) && !img.src) {
      img.src = img.dataset.src;
    }
  });
});
```

Problems with this approach:

```
1. FORCED LAYOUT: getBoundingClientRect() forces synchronous layout
   - Called for every element on every scroll event
   - At 100 images, 60fps scroll: 100 × 60 = 6,000 layout calculations/second
   - Main thread is saturated — animation and input lag

2. CONTINUOUS EXECUTION: fires on every scroll pixel
   - scroll event fires 60+ times/second while scrolling
   - Even tiny scrolls trigger full element traversal
   - No way to know "is this element's visibility actually different?"

3. INCORRECT: scroll event doesn't cover all visibility changes
   - Resizing viewport changes visibility without scroll
   - Zooming changes visibility without scroll
   - DOM changes change visibility without scroll
```

### The New Way — IntersectionObserver

```javascript
// ✅ IntersectionObserver: no forced layout, async, efficient
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;
        observer.unobserve(img); // stop observing after loading
      }
    });
  },
  {
    rootMargin: "200px", // start loading 200px before visible
  },
);

images.forEach((img) => observer.observe(img));

// Benefits:
// - Zero getBoundingClientRect calls (browser computes intersection natively)
// - No scroll event listener
// - Asynchronous — doesn't block paint
// - Fires only when intersection state actually changes
// - Handles resize, zoom, DOM changes automatically
```

---

## 2. The IntersectionObserver API

```javascript
const observer = new IntersectionObserver(
  callback,   // function called when intersection changes
  options     // optional configuration
);

// Options:
{
  root:       null,   // null = viewport; or a scrollable container element
  rootMargin: '0px',  // margin around root (like CSS margin: top right bottom left)
  threshold:  0,      // 0.0–1.0 or array: when to fire (fraction of element visible)
}

// Methods:
observer.observe(element);    // start observing an element
observer.unobserve(element);  // stop observing an element
observer.disconnect();        // stop observing all elements
observer.takeRecords();       // get pending undelivered entries
```

### The Callback

```javascript
const observer = new IntersectionObserver((entries, observer) => {
  // entries: array of IntersectionObserverEntry
  //   - one entry per observed element that changed intersection state
  //   - NOT called for elements that didn't change (efficient!)

  // observer: the IntersectionObserver instance (for self-reference)

  entries.forEach((entry) => {
    processEntry(entry);
  });
});
```

---

## 3. IntersectionObserverEntry — Understanding the Data

```javascript
// Each entry describes one element's intersection state at a point in time:
const entry = {
  // BOOLEAN: is any part of the element intersecting the root?
  isIntersecting: true,

  // NUMBER (0.0–1.0): fraction of element currently visible
  intersectionRatio: 0.75,

  // DOMRectReadOnly: element's bounding rect (relative to viewport)
  boundingClientRect: { top: 100, left: 0, bottom: 300, right: 320, width: 320, height: 200 },

  // DOMRectReadOnly: the actual intersection area
  intersectionRect: { top: 100, left: 0, bottom: 250, right: 320, width: 320, height: 150 },

  // DOMRectReadOnly: root's bounding rect (viewport rect if root is null)
  rootBounds: { top: 0, left: 0, bottom: 900, right: 1440, width: 1440, height: 900 },

  // Element: the observed element
  target: <div class="card">,

  // DOMHighResTimeStamp: when the intersection change was observed
  time: 1234.567,
};
```

### Detecting Entry vs Exit

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      // Element ENTERED the viewport (or threshold was crossed entering)
      entry.target.classList.add("visible");
    } else {
      // Element EXITED the viewport (or threshold was crossed exiting)
      entry.target.classList.remove("visible");
    }
  });
});
```

### Detecting Scroll Direction

```javascript
// IntersectionObserver doesn't directly tell you scroll direction
// But you can infer it from the entry:

const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    const rect = entry.boundingClientRect;

    if (!entry.isIntersecting) {
      // Element left viewport. Which direction?
      if (rect.top < 0) {
        // element top is above viewport → scrolled DOWN past it
        console.log("scrolled past (down)");
      } else {
        // element top is below viewport → scrolled UP past it
        console.log("scrolled past (up)");
      }
    }
  });
});
```

---

## 4. threshold — Controlling When Callbacks Fire

`threshold` defines the fraction(s) of element visibility at which the callback fires.

```javascript
// threshold: 0 (default)
// Fires when ANY pixel of element enters/exits viewport
{
  threshold: 0;
}

// threshold: 0.5
// Fires when 50% of element is visible (and when it drops below 50%)
{
  threshold: 0.5;
}

// threshold: 1.0
// Fires only when element is FULLY visible (and when fully hidden)
{
  threshold: 1.0;
}

// threshold: array — fires at each crossing
{
  threshold: [0, 0.25, 0.5, 0.75, 1.0];
}
// Fires when: 0%, 25%, 50%, 75%, 100% visible — and again when crossing back
```

### Threshold Use Cases

```javascript
// Lazy loading: fire immediately when any pixel enters (threshold = 0, default)
{
  threshold: 0;
}

// Animation: fire when element is mostly visible (not partially)
{
  threshold: 0.3;
} // 30% visible before animating in

// Progress tracking: track visibility percentage
{
  threshold: Array.from({ length: 101 }, (_, i) => i / 100);
}
// Fires at every 1% visibility change — use for reading progress bars

// Completion event: fire only when fully visible
{
  threshold: 1.0;
}
// Use for: "was this article fully read?" tracking
```

---

## 5. rootMargin — Expanding and Contracting the Root

`rootMargin` extends or shrinks the root's bounding box, affecting when elements are considered "intersecting." It follows CSS margin shorthand: `"top right bottom left"`.

```javascript
// Expand root downward by 200px:
// Elements are "visible" 200px BEFORE they enter the actual viewport
{
  rootMargin: "200px";
}
// Use for: lazy loading (start loading before element is visible)

// Expand in all directions:
{
  rootMargin: "100px";
}
// Fires 100px before AND after entering/exiting viewport

// Shrink root — create a "trigger line" in the middle of the viewport:
{
  rootMargin: "-50% 0px -50% 0px";
}
// Element must be in the middle 0% of the viewport (horizontal line)
// Use for: "active section" in table of contents

// Pre-load with generous margin, fire animation with none:
// (requires two observers)
const preloadObserver = new IntersectionObserver(fn, { rootMargin: "500px" });
const animationObserver = new IntersectionObserver(fn, { rootMargin: "0px" });
```

### rootMargin Visual

```
rootMargin: '200px 0px 0px 0px'

┌─────────────────────────────────────────┐ ← virtual top (200px above actual)
│ 200px expansion                         │
├─────────────────────────────────────────┤ ← actual viewport top
│                                         │
│           VIEWPORT                      │
│                                         │
├─────────────────────────────────────────┤ ← actual viewport bottom
└─────────────────────────────────────────┘

An element scrolling up: when its top edge reaches the virtual top
(200px above viewport), the observer fires — 200px before it's actually visible
```

---

## 6. Lazy Image Loading

The canonical IntersectionObserver use case:

```javascript
// Lazy image loading — full implementation
class LazyImageLoader {
  #observer;

  constructor(options = {}) {
    const {
      rootMargin = "200px 0px", // load 200px before entering viewport
      threshold = 0,
      selector = "img[data-src]",
    } = options;

    this.#observer = new IntersectionObserver(this.#onIntersection.bind(this), {
      rootMargin,
      threshold,
    });

    // Observe all matching images
    document.querySelectorAll(selector).forEach((img) => {
      this.#observer.observe(img);
    });
  }

  #onIntersection(entries) {
    entries.forEach((entry) => {
      if (!entry.isIntersecting) return;

      const img = entry.target;
      this.#loadImage(img);
      this.#observer.unobserve(img); // stop observing — only load once
    });
  }

  #loadImage(img) {
    const src = img.dataset.src;
    const srcset = img.dataset.srcset;
    const sizes = img.dataset.sizes;

    if (!src && !srcset) return;

    // Create a new image to preload
    const tempImg = new Image();
    tempImg.onload = () => {
      if (src) img.src = src;
      if (srcset) img.srcset = srcset;
      if (sizes) img.sizes = sizes;

      img.removeAttribute("data-src");
      img.removeAttribute("data-srcset");
      img.classList.add("loaded");
    };
    tempImg.onerror = () => {
      img.classList.add("error");
    };
    if (srcset) tempImg.srcset = srcset;
    if (src) tempImg.src = src;
  }

  // Observe dynamically added images
  observe(element) {
    this.#observer.observe(element);
  }

  destroy() {
    this.#observer.disconnect();
  }
}

// Usage
const loader = new LazyImageLoader({ rootMargin: "300px 0px" });
```

### HTML Pattern

```html
<!-- ✅ Native lazy loading (simplest) -->
<img src="image.jpg" loading="lazy" alt="Description" />

<!-- ✅ Custom lazy loading with data attributes -->
<img
  src="placeholder.svg"
  ←
  shown
  immediately
  (tiny)
  data-src="image.jpg"
  ←
  loaded
  when
  visible
  data-srcset="image-400.jpg 400w, image-800.jpg 800w"
  data-sizes="(max-width: 400px) 400px, 800px"
  alt="Description"
  class="lazy-image"
  width="800"
  ←
  prevents
  layout
  shift
  height="600"
  ←
  prevents
  CLS
/>
```

### Blur-Up Technique (LQIP)

```javascript
// Low Quality Image Placeholder → blur up to full quality
const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (!entry.isIntersecting) return;

    const img = entry.target;
    const full = new Image();

    full.onload = () => {
      img.src = img.dataset.src;
      img.classList.add("unblurred"); // CSS: transition filter 0.3s
    };
    full.src = img.dataset.src;

    observer.unobserve(img);
  });
});

// CSS:
// .lazy-image { filter: blur(20px); transition: filter 0.4s ease; }
// .lazy-image.unblurred { filter: blur(0); }
```

---

## 7. Infinite Scroll

```javascript
class InfiniteScroll {
  #observer;
  #sentinel;
  #loading = false;
  #hasMore = true;
  #page = 1;

  constructor(container, loadPage) {
    this.#container = container;
    this.#loadPage = loadPage;

    // Sentinel: invisible element at the end of the list
    this.#sentinel = document.createElement("div");
    this.#sentinel.className = "scroll-sentinel";
    this.#sentinel.style.cssText = "height: 1px; pointer-events: none;";
    container.appendChild(this.#sentinel);

    this.#observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && !this.#loading && this.#hasMore) {
          this.#loadNextPage();
        }
      },
      {
        threshold: 0,
        rootMargin: "400px", // start loading 400px before reaching the end
      },
    );

    this.#observer.observe(this.#sentinel);
  }

  async #loadNextPage() {
    this.#loading = true;
    this.#showLoader();

    try {
      const items = await this.#loadPage(this.#page);

      if (items.length === 0) {
        this.#hasMore = false;
        this.#observer.disconnect(); // no more data → stop observing
        this.#showEndMessage();
        return;
      }

      this.#renderItems(items);
      this.#page++;

      // Re-append sentinel after new items (keeps it at bottom)
      this.#container.appendChild(this.#sentinel);
    } catch (err) {
      this.#showError(err);
    } finally {
      this.#loading = false;
      this.#hideLoader();
    }
  }

  #renderItems(items) {
    const fragment = document.createDocumentFragment();
    items.forEach((item) => {
      const el = createItemElement(item);
      fragment.appendChild(el);
    });
    // Insert before sentinel
    this.#container.insertBefore(fragment, this.#sentinel);
  }

  destroy() {
    this.#observer.disconnect();
    this.#sentinel.remove();
  }
}

// Usage
const feed = new InfiniteScroll(document.getElementById("feed"), (page) =>
  api.getPosts({ page, limit: 20 }),
);
```

---

## 8. Sticky Header Detection

Detect when a header becomes sticky without scroll events:

```javascript
// Place a sentinel element just above the sticky header
// When sentinel exits viewport: header is sticky
// When sentinel enters viewport: header is not sticky

function initStickyHeader(header) {
  // Sentinel: 1px element placed above the header
  const sentinel = document.createElement("div");
  sentinel.style.cssText = `
    position: absolute;
    top: 0;
    left: 0;
    height: 1px;
    width: 100%;
    pointer-events: none;
  `;
  header.parentNode.insertBefore(sentinel, header);

  const observer = new IntersectionObserver(
    ([entry]) => {
      // When sentinel leaves viewport (scrolled past it): header is now sticky
      header.classList.toggle("is-sticky", !entry.isIntersecting);
      header.setAttribute("aria-sticky", String(!entry.isIntersecting));
    },
    {
      threshold: [0, 1], // fire on full entry/exit
    },
  );

  observer.observe(sentinel);

  return () => {
    observer.disconnect();
    sentinel.remove();
  };
}

// CSS:
// .header { position: sticky; top: 0; transition: box-shadow 0.2s; }
// .header.is-sticky { box-shadow: 0 2px 8px rgba(0,0,0,0.15); }
```

---

## 9. Animation on Scroll

Animate elements when they scroll into view:

```javascript
// Scroll animation system
function initScrollAnimations(options = {}) {
  const {
    selector = "[data-animate]",
    rootMargin = "-10% 0px -10% 0px", // fire when 10% into viewport
    threshold = 0.1,
    once = true, // animate once (most common) or repeatedly
  } = options;

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          entry.target.classList.add("animate-in");

          if (once) {
            observer.unobserve(entry.target);
          }
        } else if (!once) {
          entry.target.classList.remove("animate-in");
        }
      });
    },
    { rootMargin, threshold },
  );

  document.querySelectorAll(selector).forEach((el) => observer.observe(el));
  return () => observer.disconnect();
}
```

```html
<!-- HTML: elements with animation type -->
<div data-animate="fade-up">
  <h2>Section Title</h2>
  <p>Content that fades up when scrolled into view.</p>
</div>

<div data-animate="slide-in-left">
  <img src="feature.jpg" alt="Feature" />
</div>
```

```css
/* CSS: initial state + transition */
[data-animate] {
  opacity: 0;
  transition:
    opacity 0.6s ease,
    transform 0.6s ease;
}

[data-animate="fade-up"] {
  transform: translateY(30px);
}

[data-animate="slide-in-left"] {
  transform: translateX(-50px);
}

/* Animated state */
[data-animate].animate-in {
  opacity: 1;
  transform: none;
}

/* Respect reduced motion */
@media (prefers-reduced-motion: reduce) {
  [data-animate] {
    opacity: 1;
    transform: none;
    transition: none;
  }
}
```

### Staggered Animations

```javascript
// Animate list items with staggered delay
const listObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (!entry.isIntersecting) return;

      const items = entry.target.querySelectorAll("[data-animate-item]");
      items.forEach((item, index) => {
        item.style.transitionDelay = `${index * 100}ms`;
        item.classList.add("animate-in");
      });

      listObserver.unobserve(entry.target);
    });
  },
  { threshold: 0.1 },
);

document.querySelectorAll("[data-animate-group]").forEach((group) => {
  listObserver.observe(group);
});
```

---

## 10. Read Progress Indicator

Track how much of an article has been read:

```javascript
function initReadProgress(article, progressBar) {
  // High-resolution threshold: fires at every 1%
  const thresholds = Array.from({ length: 101 }, (_, i) => i / 100);

  const observer = new IntersectionObserver(
    ([entry]) => {
      // intersectionRatio: how much of the article is in the viewport
      // Use this + scroll position for more accurate "read %" calculation
      const articleHeight = entry.boundingClientRect.height;
      const viewportHeight = entry.rootBounds?.height ?? window.innerHeight;
      const articleFromTop =
        entry.rootBounds.height - entry.intersectionRect.top;
      const readPercent = Math.max(
        0,
        Math.min(100, (articleFromTop / articleHeight) * 100),
      );

      progressBar.style.width = `${readPercent}%`;
      progressBar.setAttribute(
        "aria-valuenow",
        String(Math.round(readPercent)),
      );
    },
    {
      threshold: thresholds,
    },
  );

  observer.observe(article);
  return () => observer.disconnect();
}
```

---

## 11. Pausing Off-Screen Animations and Videos

Stop expensive operations when elements aren't visible:

```javascript
// Pause canvas animation when off-screen
function createAnimatedCanvas(canvas, drawFn) {
  let animationId = null;
  let isVisible = false;

  const observer = new IntersectionObserver(
    ([entry]) => {
      isVisible = entry.isIntersecting;

      if (isVisible && !animationId) {
        // Resume animation
        const loop = (timestamp) => {
          drawFn(canvas.getContext("2d"), timestamp);
          if (isVisible) {
            animationId = requestAnimationFrame(loop);
          }
        };
        animationId = requestAnimationFrame(loop);
      } else if (!isVisible && animationId) {
        // Pause animation — element not visible, no need to render
        cancelAnimationFrame(animationId);
        animationId = null;
      }
    },
    { threshold: 0 },
  );

  observer.observe(canvas);
  return () => {
    observer.disconnect();
    if (animationId) cancelAnimationFrame(animationId);
  };
}

// Auto-play/pause video
function initAutoPlayVideo(video) {
  const observer = new IntersectionObserver(
    ([entry]) => {
      if (entry.isIntersecting && entry.intersectionRatio >= 0.5) {
        video.play().catch(() => {}); // play fails silently if blocked
      } else {
        video.pause();
      }
    },
    { threshold: [0, 0.5] }, // fire at 0% and 50% visibility
  );

  observer.observe(video);
  return () => observer.disconnect();
}
```

---

## 12. Ad Impression Tracking

IAB standards require an ad to be 50% visible for at least 1 second to count as a viewable impression:

```javascript
class ImpressionTracker {
  #observer;
  #timers = new Map(); // element → timer ID

  constructor(selector = "[data-ad-id]") {
    this.#observer = new IntersectionObserver(this.#onIntersection.bind(this), {
      threshold: 0.5, // 50% visible required (IAB standard)
    });

    document.querySelectorAll(selector).forEach((ad) => {
      this.#observer.observe(ad);
    });
  }

  #onIntersection(entries) {
    entries.forEach((entry) => {
      const ad = entry.target;
      const id = ad.dataset.adId;

      if (entry.isIntersecting) {
        // Start 1-second timer
        if (!this.#timers.has(id)) {
          const timer = setTimeout(() => {
            this.#recordImpression(id);
            this.#timers.delete(id);
            this.#observer.unobserve(ad); // recorded — stop observing
          }, 1000);

          this.#timers.set(id, timer);
        }
      } else {
        // Ad left view before 1 second — cancel timer
        const timer = this.#timers.get(id);
        if (timer) {
          clearTimeout(timer);
          this.#timers.delete(id);
        }
      }
    });
  }

  #recordImpression(adId) {
    analytics.track("ad_impression", {
      adId,
      timestamp: Date.now(),
    });
  }

  destroy() {
    this.#timers.forEach(clearTimeout);
    this.#timers.clear();
    this.#observer.disconnect();
  }
}
```

---

## 13. One Observer vs Many

One observer can observe many elements — creating one per element is wasteful.

```javascript
// ❌ One observer per element — N IntersectionObserver instances
elements.forEach((el) => {
  const observer = new IntersectionObserver(([entry]) => {
    handleEntry(entry);
  });
  observer.observe(el);
});
// N observers, N callback closures, N native objects

// ✅ One observer, many elements
const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => handleEntry(entry));
}, options);

elements.forEach((el) => observer.observe(el));
// One observer handles all N elements
// entries array contains only elements that changed state (efficient)
```

### Observer Pool for Different Options

```javascript
// When you need different options for different element groups:
const lazyLoadObserver = new IntersectionObserver(handleLazyLoad, {
  rootMargin: "300px",
  threshold: 0,
});

const animationObserver = new IntersectionObserver(handleAnimation, {
  rootMargin: "-10%",
  threshold: 0.1,
});

const impressionObserver = new IntersectionObserver(handleImpression, {
  threshold: 0.5,
});

// Each observer has a clear purpose — still only 3 observers for the entire page
```

---

## 14. Combining with ResizeObserver

Some use cases need both — when elements change size AND position matters:

```javascript
// Sticky sidebar: becomes sticky at a position AND needs to know sidebar height
class StickySidebar {
  #io; // IntersectionObserver
  #ro; // ResizeObserver

  constructor(sidebar, sentinel) {
    // IntersectionObserver: detect when we've scrolled to sticky point
    this.#io = new IntersectionObserver(([entry]) => {
      sidebar.classList.toggle("is-sticky", !entry.isIntersecting);
    });
    this.#io.observe(sentinel);

    // ResizeObserver: update sidebar height constraint when content changes
    this.#ro = new ResizeObserver(([entry]) => {
      const { height } = entry.contentRect;
      sidebar.style.maxHeight = `${window.innerHeight - 20}px`; // 20px padding
    });
    this.#ro.observe(sidebar);
  }

  destroy() {
    this.#io.disconnect();
    this.#ro.disconnect();
  }
}
```

---

## 15. Good Practices

### ✅ Unobserve after handling (for one-time events)

```javascript
// ✅ Stop observing once action is taken — no ongoing overhead
observer.observe(img);

new IntersectionObserver(([entry]) => {
  if (entry.isIntersecting) {
    loadImage(entry.target);
    observer.unobserve(entry.target); // ← critical: prevents future callbacks
  }
}).observe(img);
```

### ✅ Disconnect when component is destroyed

```javascript
// ✅ Clean up observers in component cleanup
class Component {
  #observer = new IntersectionObserver(this.#handler.bind(this));

  mount(element) {
    this.#observer.observe(element);
  }

  destroy() {
    this.#observer.disconnect(); // removes all observations
  }
}
```

### ✅ Use `rootMargin` to pre-load before viewport entry

```javascript
// ✅ Load resources before user sees them — no visible delay
const observer = new IntersectionObserver(load, {
  rootMargin: "400px 0px", // load 400px before entry
});
```

### ✅ Respect `prefers-reduced-motion`

```javascript
// ✅ Don't animate if user prefers reduced motion
const prefersReducedMotion = window.matchMedia(
  "(prefers-reduced-motion: reduce)",
).matches;

const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      if (prefersReducedMotion) {
        entry.target.classList.add("visible"); // just show, no animation
      } else {
        entry.target.classList.add("animate-in"); // full animation
      }
      observer.unobserve(entry.target);
    }
  });
});
```

---

## 16. Bad Practices

### ❌ Using IntersectionObserver for exact pixel positions

```javascript
// ❌ IO is not designed for pixel-accurate positioning
// Callbacks are asynchronous — position may have changed by the time callback fires
// For pixel-accurate layout work: use ResizeObserver + getBoundingClientRect
const observer = new IntersectionObserver(([entry]) => {
  // Don't use entry.boundingClientRect for layout calculations
  // It reflects state at time of intersection change, not right now
  positionTooltip(entry.boundingClientRect.right); // may be stale
});

// ✅ For positioning: use ResizeObserver + CSS position
```

### ❌ Very fine-grained thresholds on many elements

```javascript
// ❌ 101 thresholds × 1000 elements = massive callback overhead
const thresholds = Array.from({ length: 101 }, (_, i) => i / 100);
const observer = new IntersectionObserver(fn, { threshold: thresholds });
elements.forEach((el) => observer.observe(el)); // 1000 elements!
// Browser fires callback for each threshold crossing on each element

// ✅ Fine thresholds for a single article progress tracker
// NOT for 1000 list items
```

### ❌ Not handling the initial call

```javascript
// IntersectionObserver fires immediately for elements already in viewport
// Don't assume the first call is always from scrolling

const observer = new IntersectionObserver(([entry]) => {
  // ❌ Assuming this is a NEW intersection (may be initial state)
  if (entry.isIntersecting) {
    analytics.track("item_seen_for_first_time"); // fires on page load too!
  }
});

// ✅ Track whether we've seen the initial state
let initialized = false;
const observer = new IntersectionObserver(([entry]) => {
  if (!initialized) {
    initialized = true;
    return;
  } // skip initial call
  if (entry.isIntersecting) {
    analytics.track("item_scrolled_into_view");
  }
});
```

---

## 17. Common Mistakes

### Mistake 1 — Creating IntersectionObserver inside a loop

```javascript
// ❌ N separate observers
elements.forEach((el) => {
  new IntersectionObserver(callback).observe(el);
});

// ✅ One observer
const observer = new IntersectionObserver(callback);
elements.forEach((el) => observer.observe(el));
```

### Mistake 2 — Incorrect rootMargin units

```javascript
// ❌ Percentages in rootMargin are relative to root size (often counterintuitive)
{
  rootMargin: "10%";
} // 10% of viewport height — unpredictable on different screens

// ✅ Use absolute pixel values for predictable behavior
{
  rootMargin: "200px";
}
{
  rootMargin: "200px 0px";
} // top/bottom: 200px, left/right: 0px
```

### Mistake 3 — Not using threshold with isIntersecting check

```javascript
// When threshold is 0 (default), isIntersecting is true even when
// intersectionRatio is 0 (element just touching the edge)

const observer = new IntersectionObserver(
  ([entry]) => {
    // entry.isIntersecting can be true even with intersectionRatio = 0
    // (just touching the viewport edge)
    if (entry.intersectionRatio > 0) {
      // Element actually has pixels in the viewport
      handleVisible(entry.target);
    }
  },
  { threshold: 0 },
);
```

### Mistake 4 — Using `takeRecords()` without understanding it

```javascript
// takeRecords() returns pending observations that haven't been delivered
// to the callback yet, clearing the queue
// Use it before disconnecting to catch final state

const finalEntries = observer.takeRecords();
observer.disconnect();
// Process finalEntries before disconnecting
```

---

## 18. Interview-Level Explanation

> **"What is IntersectionObserver? How does it differ from scroll events for visibility detection?"**

**Strong answer:**

> "IntersectionObserver is a browser API that asynchronously reports when an element enters or exits another element or the viewport. You register a callback that fires only when an element's intersection with the root changes, not on every scroll event.
>
> The key difference from scroll-event-based visibility detection is performance. The old approach called `getBoundingClientRect()` on every element on every scroll event — which forces a synchronous layout recalculation each time. At 60fps scroll with 100 elements, that's 6,000 forced layouts per second — enough to saturate the main thread. IntersectionObserver delegates intersection calculation to the browser itself, which computes it off the main thread and delivers results asynchronously, batched, only when state actually changes.
>
> The API has three key configuration options: `root` specifies what container to observe intersection with (null means the viewport); `rootMargin` expands or contracts the root's effective boundary (useful for pre-loading images 300px before they're visible); and `threshold` is a fraction or array of fractions — 0 fires when any pixel intersects, 1.0 fires only when fully visible.
>
> The most common use cases are lazy loading images (observe with rootMargin '300px', unobserve on first intersection), infinite scroll (observe a sentinel element at the bottom of the list), scroll-triggered animations (observe sections with a threshold of 0.1), and ad impression tracking (observe with threshold 0.5, start a 1-second timer, track if element stays visible).
>
> The main gotcha is that it fires immediately on observation with the current state — you need to handle the initial callback correctly if you only want to track 'newly entered' events. And one observer should handle many elements — creating one observer per element defeats much of the performance benefit."

---

## 19. Exercises

### Exercise 1 — Build a lazy video loader

Implement a class that:

- Observes `<video data-src="...">` elements
- Loads video src when 50% visible (threshold: 0.5)
- Plays video when 70% visible
- Pauses when less than 30% visible

```javascript
class LazyVideoController {
  constructor(selector = "video[data-src]") {
    /* ... */
  }
  destroy() {
    /* ... */
  }
}
```

<details>
<summary>Solution</summary>

```javascript
class LazyVideoController {
  #loadObserver;
  #playObserver;
  #pauseObserver;

  constructor(selector = "video[data-src]") {
    // Load when 50% visible
    this.#loadObserver = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (!entry.isIntersecting) return;
          const video = entry.target;
          if (!video.src && video.dataset.src) {
            video.src = video.dataset.src;
            video.removeAttribute("data-src");
          }
          this.#loadObserver.unobserve(video); // load once
        });
      },
      { threshold: 0.5 },
    );

    // Play when 70% visible
    this.#playObserver = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting && entry.target.src) {
            entry.target.play().catch(() => {});
          }
        });
      },
      { threshold: 0.7 },
    );

    // Pause when less than 30% visible
    this.#pauseObserver = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (!entry.isIntersecting) {
            entry.target.pause();
          }
        });
      },
      { threshold: 0.3 },
    );

    document.querySelectorAll(selector).forEach((video) => {
      this.#loadObserver.observe(video);
      this.#playObserver.observe(video);
      this.#pauseObserver.observe(video);
    });
  }

  destroy() {
    this.#loadObserver.disconnect();
    this.#playObserver.disconnect();
    this.#pauseObserver.disconnect();
  }
}
```

</details>

---

### Exercise 2 — Table of contents with active section

Implement a table of contents that highlights the currently-reading section:

```html
<nav id="toc">
  <a href="#intro">Introduction</a>
  <a href="#setup">Setup</a>
  <a href="#usage">Usage</a>
</nav>

<section id="intro">...</section>
<section id="setup">...</section>
<section id="usage">...</section>
```

<details>
<summary>Solution</summary>

```javascript
function initTableOfContents() {
  const sections = document.querySelectorAll("section[id]");
  const tocLinks = document.querySelectorAll('#toc a[href^="#"]');

  const sectionMap = new Map(
    [...sections].map((section) => [section.id, section]),
  );

  // Track which sections are currently visible
  const visible = new Set();

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          visible.add(entry.target.id);
        } else {
          visible.delete(entry.target.id);
        }
      });

      // Highlight the first visible section
      // (topmost section currently in view)
      const sectionIds = [...sections].map((s) => s.id);
      const activeId = sectionIds.find((id) => visible.has(id));

      tocLinks.forEach((link) => {
        const id = link.getAttribute("href").slice(1);
        link.classList.toggle("active", id === activeId);
        link.setAttribute("aria-current", id === activeId ? "true" : "false");
      });
    },
    {
      // -15% top: section must be 15% from top before counting as active
      // -70% bottom: section stops being active before it fully scrolls out
      rootMargin: "-15% 0px -70% 0px",
      threshold: 0,
    },
  );

  sections.forEach((section) => observer.observe(section));

  return () => observer.disconnect();
}
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md) — Observer pattern fundamentals
- [`performance/02-virtualization-windowing.md`](./02-virtualization-windowing.md) — IntersectionObserver for virtual scrolling
- [`performance/04-raf-optimization.md`](./04-raf-optimization.md) — Pausing animations for off-screen elements
- [`browser-internals/08-critical-rendering-path.md`](../browser-internals/08-critical-rendering-path.md) — Lazy loading to improve CRP

---

<div align="center">

**Next:** [`performance/10-canvas-optimization.md`](./10-canvas-optimization.md) →

</div>
