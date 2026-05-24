# 01 — DOM Optimization

> **"The DOM is not your code's natural habitat — it's an expensive foreign environment across a bridge. Every unnecessary crossing costs you. Every unnecessary node costs you memory. The fastest DOM operation is the one you never perform."**

The DOM is one of the most expensive APIs in frontend development. Not because it's poorly designed, but because it lives in the browser's rendering engine (C++ land), and every interaction from JavaScript crosses a language boundary. Understanding this cost — and how to minimize it — separates engineers who write code that works from engineers who write code that stays fast under load.

---

## 📚 Table of Contents

1. [The JavaScript-to-DOM Bridge](#1-the-javascript-to-dom-bridge)
2. [DOM Node Cost](#2-dom-node-cost)
3. [Querying the DOM Efficiently](#3-querying-the-dom-efficiently)
4. [DOM Manipulation — Minimizing Updates](#4-dom-manipulation--minimizing-updates)
5. [DocumentFragment — Off-DOM Building](#5-documentfragment--off-dom-building)
6. [innerHTML vs createElement — When to Use Each](#6-innerhtml-vs-createelement--when-to-use-each)
7. [Cloning Nodes](#7-cloning-nodes)
8. [Reading DOM Properties Safely](#8-reading-dom-properties-safely)
9. [Event Delegation for Large Lists](#9-event-delegation-for-large-lists)
10. [DOM Tree Depth and Width](#10-dom-tree-depth-and-width)
11. [Attribute vs Property vs Style](#11-attribute-vs-property-vs-style)
12. [Dataset and Custom Data Attributes](#12-dataset-and-custom-data-attributes)
13. [DOM Recycling](#13-dom-recycling)
14. [Benchmarking DOM Operations](#14-benchmarking-dom-operations)
15. [Good Practices](#15-good-practices)
16. [Bad Practices](#16-bad-practices)
17. [Common Mistakes](#17-common-mistakes)
18. [Interview-Level Explanation](#18-interview-level-explanation)
19. [Exercises](#19-exercises)

---

## 1. The JavaScript-to-DOM Bridge

JavaScript and the DOM live in separate worlds:

```
┌─────────────────────────────────────────────────────────────────┐
│  JavaScript Engine (V8)                                          │
│  Your code runs here.                                            │
│  Objects, closures, async — all in V8's heap.                   │
└────────────────────────────┬────────────────────────────────────┘
                             │  DOM Bridge (binding layer)
                             │  Each DOM call crosses this bridge
                             │  ~300ns overhead per crossing
                             │  (trivial once, expensive × millions)
┌────────────────────────────┴────────────────────────────────────┐
│  Browser Rendering Engine (Blink/WebKit/Gecko — C++)            │
│  DOM tree, style engine, layout engine live here.               │
│  Every querySelector, appendChild, offsetWidth crosses above.   │
└─────────────────────────────────────────────────────────────────┘
```

### The Bridge Cost in Numbers

```javascript
// Each of these is ONE bridge crossing:
document.getElementById("app"); // ~300ns
element.appendChild(child); // ~500ns (may trigger layout)
element.style.color = "red"; // ~400ns
element.offsetWidth; // ~1000ns+ (forces layout)
element.getBoundingClientRect(); // ~2000ns (forces layout)

// In a tight loop of 10,000 operations:
// 300ns × 10,000 = 3ms  (acceptable)
// 1000ns × 10,000 = 10ms (one frame budget gone)
// 2000ns × 10,000 = 20ms (janky!)

// Real cost comes from WHAT each operation triggers,
// not just the bridge crossing itself
```

---

## 2. DOM Node Cost

Every DOM node has a memory footprint and a rendering cost. Understanding the cost helps you minimize unnecessary nodes.

### Memory per Node Type

```
Approximate memory footprint (V8 + Blink):

Element node (<div>, <p>, etc.):   ~500-1000 bytes
  - C++ Node object: ~200 bytes
  - JavaScript wrapper: ~100 bytes
  - Style data: ~100 bytes
  - Layout box: ~100 bytes
  - Miscellaneous: ~100-500 bytes

Text node ("Hello World"):         ~100 bytes

Comment node (<!-- -->):           ~80 bytes

Document fragment (off-DOM):       ~50 bytes
```

### The DOM Node Count Problem

```
10 items list: 10 × 1000 bytes = 10KB
100 items list: 100 × 1000 bytes = 100KB
10,000 items list: 10,000 × 1000 bytes = 10MB (significant!)

Plus:
  Style matching: O(n) elements × O(m) CSS rules
  Layout: O(n) for simple, can be O(n²) for table/flex
  Paint: proportional to visible area
  Event bubbling: depth of the tree
```

### Checking DOM Node Count

```javascript
// Count DOM nodes (for monitoring)
function countNodes(root = document) {
  let count = 0;
  const walker = document.createTreeWalker(root, NodeFilter.SHOW_ALL);
  while (walker.nextNode()) count++;
  return count;
}

// Rule of thumb: > 1500 nodes is a warning sign
// > 3000 nodes often causes perceptible rendering slowness
// Modern benchmarks: ideal < 1000 nodes for 60fps on mid-range devices
```

---

## 3. Querying the DOM Efficiently

### Query Method Costs

```javascript
// FAST — O(1) using internal hash maps
document.getElementById("nav");
document.getElementsByTagName("div"); // returns HTMLCollection (live!)
document.getElementsByClassName("card"); // returns HTMLCollection (live!)

// MODERATE — CSS selector engine
document.querySelector(".card"); // first match
document.querySelectorAll(".card"); // all matches (static NodeList)

// SLOW — complex selectors, large DOM
document.querySelectorAll(".list .item:nth-child(odd) span");
// For every span: walk up checking nth-child, then item, then list
```

### Live vs Static Collections

```javascript
// LIVE collection — updates when DOM changes
const liveCollection = document.getElementsByTagName("li");
// liveCollection.length changes if you add/remove <li> elements
// Useful when you need to track DOM changes
// DANGEROUS in loops: can cause infinite loops

// STATIC NodeList — snapshot at query time
const staticList = document.querySelectorAll("li");
// staticList.length stays the same even if DOM changes
// Generally preferred for predictability

// ❌ Dangerous with live collection:
const items = document.getElementsByTagName("li");
for (let i = 0; i < items.length; i++) {
  items[i].remove(); // removing shifts indices! items[0] changes each iteration
  // Only odd-indexed elements are removed
}

// ✅ Safe: convert to array first or use static NodeList
const items = [...document.querySelectorAll("li")]; // static snapshot
items.forEach((item) => item.remove()); // safe: array doesn't update
```

### Caching Query Results

```javascript
// ❌ Querying on every use — redundant work
function updateUI() {
  document.getElementById("counter").textContent = count++;
  document.getElementById("counter").classList.toggle("high", count > 10);
  document.getElementById("counter").style.color = count > 20 ? "red" : "";
}

// ✅ Cache the reference — one query, many uses
const counter = document.getElementById("counter");
function updateUI() {
  counter.textContent = count++;
  counter.classList.toggle("high", count > 10);
  counter.style.color = count > 20 ? "red" : "";
}
```

### Scoped Queries

```javascript
// ❌ Querying from document root — searches entire DOM
const items = document.querySelectorAll(".item");

// ✅ Scope to a container — searches only within container
const container = document.getElementById("list");
const items = container.querySelectorAll(".item");
// Faster for large DOMs — fewer nodes to traverse
```

---

## 4. DOM Manipulation — Minimizing Updates

### The Batch Principle

Every DOM write that changes layout must trigger at minimum a style recalculation. Batch them to minimize recalculations.

```javascript
// ❌ Three separate mutations = three potential layout invalidations
element.style.width = "200px";
element.style.height = "100px";
element.style.margin = "10px";

// ✅ Option 1: CSS class — one recalculation
element.classList.add("sized"); // .sized { width: 200px; height: 100px; margin: 10px; }

// ✅ Option 2: cssText — one DOM write
element.style.cssText = "width: 200px; height: 100px; margin: 10px;";

// ✅ Option 3: CSS custom properties — one root update
document.documentElement.style.setProperty("--card-size", "200px");
// All elements using var(--card-size) update automatically
```

### Minimizing DOM Insertions

```javascript
// ❌ Each appendChild triggers potential reflow
const list = document.getElementById("list");
for (const item of data) {
  const li = document.createElement("li");
  li.textContent = item.name;
  list.appendChild(li); // N separate DOM mutations
}

// ✅ Fragment: build off-DOM, insert once
const fragment = document.createDocumentFragment();
for (const item of data) {
  const li = document.createElement("li");
  li.textContent = item.name;
  fragment.appendChild(li); // no DOM mutation, no reflow
}
list.appendChild(fragment); // one DOM mutation, one potential reflow
```

### Replacing Content Efficiently

```javascript
// ❌ Removing then re-adding all children — N remove + N add
function refreshList(data) {
  const list = document.getElementById("list");
  while (list.firstChild) {
    list.removeChild(list.firstChild); // N DOM mutations
  }
  data.forEach((item) => {
    const li = document.createElement("li");
    li.textContent = item.name;
    list.appendChild(li); // N more mutations
  });
}

// ✅ Option 1: replaceChildren (modern)
function refreshList(data) {
  const list = document.getElementById("list");
  const newChildren = data.map((item) => {
    const li = document.createElement("li");
    li.textContent = item.name;
    return li;
  });
  list.replaceChildren(...newChildren); // single DOM mutation
}

// ✅ Option 2: innerHTML (fast for large updates, use carefully)
function refreshList(data) {
  document.getElementById("list").innerHTML = data
    .map((item) => `<li>${escapeHtml(item.name)}</li>`)
    .join("");
  // One mutation, browser parses and builds subtree internally
}
```

---

## 5. DocumentFragment — Off-DOM Building

A `DocumentFragment` is a lightweight container that exists off the DOM. Manipulating it doesn't trigger layout or paint. When you append it to the DOM, its children are moved (not the fragment itself), resulting in one layout invalidation.

```javascript
// DocumentFragment: off-DOM container
const frag = document.createDocumentFragment();

// Build your tree in the fragment — zero layout cost
for (let i = 0; i < 1000; i++) {
  const div = document.createElement("div");
  div.className = "item";
  div.textContent = `Item ${i}`;

  const btn = document.createElement("button");
  btn.textContent = "Click";
  div.appendChild(btn);

  frag.appendChild(div);
}

// Insert all 1000 items at once — ONE layout invalidation
container.appendChild(frag);
// Note: frag is now empty (children moved to container, not copied)
```

### When Fragment Makes a Difference

```javascript
// Benchmark: 1000 direct appends vs fragment
// (run in DevTools console)

function directAppend(container, count) {
  const start = performance.now();
  for (let i = 0; i < count; i++) {
    const div = document.createElement("div");
    div.textContent = i;
    container.appendChild(div); // layout invalidated each iteration
  }
  return performance.now() - start;
}

function fragmentAppend(container, count) {
  const start = performance.now();
  const frag = document.createDocumentFragment();
  for (let i = 0; i < count; i++) {
    const div = document.createElement("div");
    div.textContent = i;
    frag.appendChild(div); // off-DOM, no layout
  }
  container.appendChild(frag); // one layout invalidation
  return performance.now() - start;
}

// Typical results (count = 1000):
// Direct:   8–25ms
// Fragment: 3–12ms
// Savings: 30–60% — more significant on slow devices
```

---

## 6. innerHTML vs createElement — When to Use Each

Both create DOM from content, but with different characteristics.

### `innerHTML` — HTML String Parsing

```javascript
// innerHTML: parse an HTML string, replace children
element.innerHTML = "<ul><li>A</li><li>B</li></ul>";

// Pros:
//  - Fast for large, complex HTML structures
//  - Concise code
//  - Browser's HTML parser handles structure

// Cons:
//  - XSS risk if data is not escaped
//  - Destroys existing DOM — all event listeners lost
//  - Can't reuse existing elements (all re-created)
//  - Slightly slower for simple elements (parse overhead)
```

### `createElement` — Programmatic Node Creation

```javascript
// createElement: build DOM nodes imperatively
const ul = document.createElement("ul");
["A", "B"].forEach((text) => {
  const li = document.createElement("li");
  li.textContent = text; // SAFE: textContent doesn't parse HTML
  ul.appendChild(li);
});
element.appendChild(ul);

// Pros:
//  - No XSS risk (textContent never interpreted as HTML)
//  - Event listeners survive (if attached before insertion)
//  - Reuse/move existing elements
//  - Precise control over every node

// Cons:
//  - Verbose for complex structures
//  - Slightly slower for deeply nested HTML (more function calls)
```

### The Security Decision

```javascript
// ❌ NEVER: user data in innerHTML — XSS vulnerability
const name = userInput; // might be: <script>alert('XSS')</script>
element.innerHTML = `<p>Hello, ${name}!</p>`; // EXECUTES THE SCRIPT

// ✅ textContent: user data as text only
const p = document.createElement("p");
p.textContent = `Hello, ${userInput}!`; // rendered as text, not HTML
element.appendChild(p);

// ✅ If you need HTML but with safe user data: escape or sanitize
function escapeHtml(str) {
  const div = document.createElement("div");
  div.textContent = str;
  return div.innerHTML; // escapes < > & " '
}
element.innerHTML = `<p>Hello, ${escapeHtml(userInput)}!</p>`;

// ✅ DOMPurify for rich HTML input that needs sanitizing
import DOMPurify from "dompurify";
element.innerHTML = DOMPurify.sanitize(richHtmlFromUser);
```

### Performance Comparison

```javascript
// When innerHTML is faster: large complex subtrees
// createElement faster: simple elements, reuse, safety required

// For 100 items with complex structure:
// innerHTML is often faster (browser's C++ parser vs N JS function calls)

// For simple items (just text):
// createElement can be comparable to innerHTML
// and is always safer
```

---

## 7. Cloning Nodes

When creating many identical or similar elements, cloning is faster than creating from scratch.

```javascript
// ❌ Creating from scratch each time
function createCard(data) {
  const card = document.createElement("div");
  card.className = "card";
  const title = document.createElement("h2");
  title.className = "card__title";
  const body = document.createElement("p");
  body.className = "card__body";
  card.appendChild(title);
  card.appendChild(body);
  return card;
}

// Creating 100 cards: 100 × (5 createElement + 2 appendChild) = 700 DOM calls

// ✅ Clone a template: one creation, N clones
const template = (() => {
  const card = document.createElement("div");
  card.className = "card";
  const title = document.createElement("h2");
  title.className = "card__title";
  const body = document.createElement("p");
  body.className = "card__body";
  card.appendChild(title);
  card.appendChild(body);
  return card;
})();

function createCard(data) {
  const card = template.cloneNode(true); // deep clone: true
  card.querySelector(".card__title").textContent = data.title;
  card.querySelector(".card__body").textContent = data.body;
  return card;
}

// Creating 100 cards: 100 × (1 cloneNode + 2 querySelector) = 300 DOM calls
// ~57% fewer DOM operations
```

### The `<template>` Element

The HTML `<template>` element is designed for exactly this use case:

```html
<!-- Define template in HTML -->
<template id="card-template">
  <div class="card">
    <h2 class="card__title"></h2>
    <p class="card__body"></p>
    <button class="card__btn">Read More</button>
  </div>
</template>
```

```javascript
const template = document.getElementById("card-template");

function createCard(data) {
  const clone = template.content.cloneNode(true); // deep clone DocumentFragment
  clone.querySelector(".card__title").textContent = data.title;
  clone.querySelector(".card__body").textContent = data.body;
  return clone;
}

const fragment = document.createDocumentFragment();
cards.forEach((data) => fragment.appendChild(createCard(data)));
container.appendChild(fragment);
```

---

## 8. Reading DOM Properties Safely

Certain DOM reads force synchronous layout recalculation. Reading them efficiently is critical.

### The Safe Read Pattern

```javascript
// ❌ Mixed reads and writes — layout thrashing
elements.forEach((el) => {
  const width = el.offsetWidth; // READ: forces layout
  el.style.width = width * 1.1 + "px"; // WRITE: invalidates
  // Next READ triggers layout again
});

// ✅ All reads, then all writes
const widths = elements.map((el) => el.offsetWidth); // all reads (one layout)
elements.forEach((el, i) => {
  el.style.width = widths[i] * 1.1 + "px"; // all writes
});
```

### Reading Without Forcing Layout

Some reads don't force layout when the layout is still clean (no writes since last layout):

```javascript
// These reads DON'T force layout if no mutations since last layout:
element.offsetWidth; // reads cached value
element.getBoundingClientRect(); // reads cached value

// But after ANY write:
element.style.width = "100px"; // invalidates layout
element.offsetWidth; // NOW forces synchronous layout

// The key: avoid reads AFTER writes in the same task
```

### ResizeObserver — Layout-Free Size Reading

```javascript
// ✅ ResizeObserver: browser provides size without forced layout
const ro = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    // width and height are provided — no getBoundingClientRect needed
    handleResize(entry.target, width, height);
  }
});

ro.observe(container);
// Fires only when element's size actually changes
// No polling, no forced layout, no scroll event
```

---

## 9. Event Delegation for Large Lists

Attaching one event listener to a parent vs one per child is one of the most impactful DOM optimizations for dynamic lists.

### The Problem with Per-Element Listeners

```javascript
// ❌ N listeners for N items
const list = document.getElementById("list");
items.forEach((item) => {
  const li = document.createElement("li");
  li.textContent = item.name;
  li.addEventListener("click", () => handleClick(item)); // one per item
  list.appendChild(li);
});

// Problems:
// - Memory: N closure allocations
// - If items are removed and re-added: must re-attach listeners
// - Dynamic items added later: must manually attach listeners
// - 1000 items = 1000 event listener objects in memory
```

### Event Delegation — One Listener for All

```javascript
// ✅ One listener handles all items via event bubbling
const list = document.getElementById("list");

list.addEventListener("click", (event) => {
  // Find the closest .list-item ancestor of the clicked element
  const item = event.target.closest(".list-item");
  if (!item) return; // click was outside any list item

  const id = item.dataset.id; // data attached to DOM node
  handleClick(id);
});

// Build the list — no event listeners on individual items
const fragment = document.createDocumentFragment();
items.forEach((item) => {
  const li = document.createElement("li");
  li.className = "list-item";
  li.dataset.id = item.id; // store data on element
  li.textContent = item.name;
  fragment.appendChild(li);
});
list.appendChild(fragment);

// Benefits:
// - ONE listener regardless of item count
// - Works automatically for dynamically added items
// - Memory: one closure instead of N
```

### `event.target` vs `event.currentTarget`

```javascript
list.addEventListener("click", (event) => {
  event.target; // the element that was ACTUALLY clicked (may be child)
  event.currentTarget; // the element the listener is on (list in this case)

  // ❌ Using target directly — may be a child element
  if (event.target.className === "list-item") {
    /* won't work for nested */
  }

  // ✅ Using closest — finds nearest ancestor matching selector
  const item = event.target.closest(".list-item");
  // Works regardless of how deeply nested the clicked element is
});
```

### Delegating Multiple Event Types

```javascript
// ✅ Delegate multiple event types efficiently
const list = document.getElementById("list");

list.addEventListener("click", handleListInteraction);
list.addEventListener("keydown", handleListInteraction);

function handleListInteraction(event) {
  // For click: any click within list
  // For keydown: keyboard navigation
  const item = event.target.closest("[data-id]");
  if (!item) return;

  if (
    event.type === "click" ||
    (event.type === "keydown" && event.key === "Enter")
  ) {
    selectItem(item.dataset.id);
  }
  if (event.type === "keydown" && event.key === "Delete") {
    deleteItem(item.dataset.id);
  }
}
```

---

## 10. DOM Tree Depth and Width

Tree structure affects rendering performance:

```
DEEP tree (bad):
  body
    div.wrapper         ← 10 levels deep
      div.container
        div.section
          div.row
            div.col
              article
                header
                  h1 "Title"  ← this is level 9

  Issues:
    - Style matching: must check all 9 ancestors for each rule
    - Layout: geometry propagation goes 9 levels
    - Event bubbling: 9 levels for any event

WIDE tree (can be bad):
  ul
    li × 10,000  ← 10,000 siblings

  Issues:
    - Style matching: O(n) for nth-child selectors
    - Layout: O(n) for flex/grid layout calculation
    - DOM traversal: slow for childNodes walks
```

### Optimal Tree Structure

```javascript
// ✅ Flat, shallow tree with virtualization for long lists
// Ideal depth: < 10 levels
// For long lists: only render visible items (virtualization)

// ❌ Deeply nested wrapper hell
<div class="outer-wrapper">
  <div class="inner-wrapper">
    <div class="content-wrapper">
      <div class="section-wrapper">
        <div class="row-wrapper">
          <p>Actual content</p>
        </div>
      </div>
    </div>
  </div>
</div>

// ✅ Flat and semantic
<section class="content">
  <p>Actual content</p>
</section>
```

---

## 11. Attribute vs Property vs Style

Three ways to set element data/appearance — with different performance characteristics:

### Attribute (`setAttribute`)

```javascript
// Attributes: stored in the HTML, serialized to the page
element.setAttribute("data-id", "42"); // stored in attribute
element.getAttribute("data-id"); // retrieves it
element.removeAttribute("data-id");

// Performance: moderate (string parsing, triggers mutation records)
// Use for: custom data (data-*), ARIA attributes, initial HTML values
// Note: attribute value is always a string
```

### Property (direct access)

```javascript
// Properties: JavaScript object properties on DOM nodes
element.id = "main"; // reflects to attribute `id`
element.className = "card"; // reflects to attribute `class`
element.hidden = true; // reflects to attribute `hidden`
element.disabled = false; // reflects `disabled` attribute

// Performance: faster than setAttribute for standard properties
// Note: property = JavaScript type; attribute = string
// element.disabled = true   → property is boolean
// element.getAttribute('disabled') → attribute is "" (empty string) or null
```

### Style

```javascript
// Inline styles: highest CSS specificity, reflected in style attribute
element.style.color = "red"; // sets inline style
element.style.cssText = "color: red; font-size: 16px;"; // batch
element.style.removeProperty("color"); // remove one property

// ✅ Prefer classList over style for visual state changes
element.classList.add("is-active"); // CSS handles the visual change
element.classList.remove("is-loading");
element.classList.toggle("is-open", condition);
element.classList.replace("state-a", "state-b");

// classList is more maintainable and respects CSS cascade
// style overrides everything (high specificity)
```

---

## 12. Dataset and Custom Data Attributes

`data-*` attributes store application data on DOM elements.

```html
<li class="item" data-id="42" data-category="electronics" data-price="29.99">
  Widget
</li>
```

```javascript
const item = document.querySelector(".item");

// Reading dataset
item.dataset.id; // "42" (always a string)
item.dataset.category; // "electronics"
item.dataset.price; // "29.99"

// Writing dataset
item.dataset.selected = "true";
// → adds data-selected="true" attribute

// Checking
"id" in item.dataset; // true
item.dataset.hasOwnProperty("id"); // true

// Deleting
delete item.dataset.selected;
// → removes data-selected attribute
```

### Dataset vs Alternative Storage

```javascript
// Option 1: dataset — stores on DOM element
li.dataset.id = productId;
// Pro: survives DOM moves, serialized to HTML
// Con: string only, visible in DOM inspection

// Option 2: WeakMap — associates data with element
const elementData = new WeakMap();
elementData.set(li, { id: productId, object: productObj });
// Pro: any type, private, GC-friendly
// Con: data lost if element is replaced (innerHTML reset)

// Option 3: Custom property on element
li._productId = productId; // informal convention
// Avoid: pollutes element namespace, may conflict with future APIs
```

---

## 13. DOM Recycling

DOM recycling (pool) reuses DOM elements instead of creating and destroying them. Critical for high-frequency list updates.

```javascript
class DOMRecycler {
  constructor(factory, initialSize = 20) {
    this._factory = factory; // function to create a new element
    this._pool = []; // available elements

    // Pre-create elements
    for (let i = 0; i < initialSize; i++) {
      this._pool.push(factory());
    }
  }

  /** Get an element (from pool or create new) */
  acquire() {
    return this._pool.length > 0 ? this._pool.pop() : this._factory();
  }

  /** Return element to pool for reuse */
  release(element) {
    // Reset element state before pooling
    element.className = "";
    element.textContent = "";
    element.removeAttribute("data-id");
    this._pool.push(element);
  }
}

// Usage in a virtualized list
const liRecycler = new DOMRecycler(() => document.createElement("li"), 50);

function updateVisibleItems(visibleData) {
  // Release all currently visible items back to pool
  currentItems.forEach((item) => {
    item.remove();
    liRecycler.release(item);
  });
  currentItems = [];

  // Acquire and populate items from pool
  const fragment = document.createDocumentFragment();
  visibleData.forEach((data) => {
    const li = liRecycler.acquire();
    li.className = "list-item";
    li.dataset.id = data.id;
    li.textContent = data.name;
    currentItems.push(li);
    fragment.appendChild(li);
  });

  container.appendChild(fragment);
}
```

---

## 14. Benchmarking DOM Operations

```javascript
// Benchmark utility for DOM operations
function benchmarkDOM(label, fn, iterations = 100) {
  // Warm up
  for (let i = 0; i < 5; i++) fn();

  const times = [];
  for (let i = 0; i < iterations; i++) {
    const start = performance.now();
    fn();
    times.push(performance.now() - start);
  }

  const avg = times.reduce((a, b) => a + b, 0) / times.length;
  const min = Math.min(...times);
  const max = Math.max(...times);

  console.log(
    `${label}: avg=${avg.toFixed(3)}ms min=${min.toFixed(3)}ms max=${max.toFixed(3)}ms`,
  );
  return { avg, min, max };
}

// Example: compare direct append vs fragment
const container = document.createElement("div");
document.body.appendChild(container);

benchmarkDOM("Direct append (100 items)", () => {
  container.innerHTML = "";
  for (let i = 0; i < 100; i++) {
    const div = document.createElement("div");
    div.textContent = i;
    container.appendChild(div);
  }
});

benchmarkDOM("Fragment append (100 items)", () => {
  container.innerHTML = "";
  const frag = document.createDocumentFragment();
  for (let i = 0; i < 100; i++) {
    const div = document.createElement("div");
    div.textContent = i;
    frag.appendChild(div);
  }
  container.appendChild(frag);
});

benchmarkDOM("innerHTML (100 items)", () => {
  container.innerHTML = Array.from(
    { length: 100 },
    (_, i) => `<div>${i}</div>`,
  ).join("");
});
```

---

## 15. Good Practices

### ✅ Cache DOM references

```javascript
// ✅ Query once, store, reuse
class Component {
  constructor(root) {
    this.el = root;
    this.title = root.querySelector(".title");
    this.body = root.querySelector(".body");
    this.btn = root.querySelector(".btn");
  }

  update(data) {
    this.title.textContent = data.title;
    this.body.textContent = data.body;
    this.btn.disabled = !data.active;
  }
}
```

### ✅ Use classList for CSS-driven state changes

```javascript
// ✅ CSS handles the visual, JS handles the state
button.classList.add("is-loading");
// CSS: .btn.is-loading { opacity: 0.6; pointer-events: none; }
// Separation of concerns + better specificity control
```

### ✅ Use DocumentFragment for batch insertions

```javascript
// ✅ Always build off-DOM, insert once
const frag = document.createDocumentFragment();
data.forEach((item) => frag.appendChild(createItem(item)));
list.appendChild(frag);
```

### ✅ Use event delegation for dynamic lists

```javascript
// ✅ One listener, works for current and future children
container.addEventListener("click", (e) => {
  const item = e.target.closest("[data-action]");
  if (item) handleAction(item.dataset.action, item.dataset.id);
});
```

---

## 16. Bad Practices

### ❌ Reading layout-forcing properties in loops

```javascript
// ❌ Forces layout per iteration
elements.forEach((el) => {
  if (el.offsetHeight > 100) el.classList.add("tall");
});

// ✅ Read all first, then write all
const heights = Array.from(elements, (el) => el.offsetHeight);
elements.forEach((el, i) => {
  if (heights[i] > 100) el.classList.add("tall");
});
```

### ❌ Repeated getElementById/querySelector in loops

```javascript
// ❌ Queries DOM on every iteration
for (let i = 0; i < 1000; i++) {
  document.getElementById("counter").textContent = i;
}

// ✅ Cache reference outside loop
const counter = document.getElementById("counter");
for (let i = 0; i < 1000; i++) {
  counter.textContent = i;
}
```

### ❌ Concatenating to innerHTML in loops

```javascript
// ❌ O(n²): string + re-parse + DOM rebuild on each iteration
let html = "";
data.forEach((item) => {
  html += `<div>${item.name}</div>`; // new string each time
  container.innerHTML = html; // re-parses entire string each time
});

// ✅ Build full string first, set once
container.innerHTML = data.map((item) => `<div>${item.name}</div>`).join("");
```

### ❌ Using innerHTML for user-provided content

```javascript
// ❌ XSS vulnerability
element.innerHTML = `<p>${userInput}</p>`;
// Always use textContent or sanitize first
```

---

## 17. Common Mistakes

### Mistake 1 — Treating innerHTML as free

```javascript
// ❌ innerHTML on every state change
function render(items) {
  list.innerHTML = items.map((i) => `<li>${i.name}</li>`).join("");
  // Destroys ALL existing DOM
  // Re-creates ALL nodes
  // Loses ALL event listeners
  // Can cause layout thrash if read immediately after
}
// Use a virtual DOM, diffing, or targeted updates instead
```

### Mistake 2 — Not cleaning up event listeners

```javascript
// ❌ Memory leak: listener not removed
function attachHandler(element) {
  element.addEventListener("click", () => {
    doSomething();
  });
  // When element is removed from DOM, listener keeps element alive
}

// ✅ Use AbortController for easy cleanup
const controller = new AbortController();
element.addEventListener("click", handler, { signal: controller.signal });
// Later:
controller.abort(); // removes all listeners using this signal
```

### Mistake 3 — `children` vs `childNodes` confusion

```javascript
const ul = document.querySelector("ul");

// childNodes: ALL nodes including text/whitespace
ul.childNodes.length; // might be 7 for 3 <li> (whitespace text nodes between)
ul.childNodes[0]; // might be a text node "\n  " (whitespace)

// children: only ELEMENT nodes
ul.children.length; // 3 (only the <li> elements)
ul.children[0]; // first <li>

// Use children for element iteration
// Use childNodes only if you need text/comment nodes
```

---

## 18. Interview-Level Explanation

> **"How do you optimize DOM operations for a high-performance frontend application?"**

**Strong answer:**

> "DOM optimization comes down to three principles: minimize the number of operations, minimize their scope, and batch reads and writes to prevent layout thrashing.
>
> The DOM bridge between JavaScript and the browser's rendering engine has overhead per crossing. The real cost isn't just the crossing itself — it's that certain DOM reads like `offsetWidth` or `getBoundingClientRect` force the browser to run a synchronous layout recalculation to return an accurate value. Alternating writes and reads in a loop — layout thrashing — can multiply layout cost by the number of iterations. The fix is to always batch all reads before all writes.
>
> For building and updating the DOM, a `DocumentFragment` lets you construct an entire subtree off-screen before inserting it. One `appendChild(fragment)` causes at most one layout invalidation instead of N. For content-heavy updates, `innerHTML` is often faster than `createElement` in a loop because the browser's C++ HTML parser is faster than N JavaScript function calls.
>
> For dynamic lists, event delegation attaches one listener to the parent container instead of one per child item. The listener uses `event.target.closest()` to find which item was clicked, even for nested elements. This reduces memory usage from O(n) listeners to O(1), and dynamically added items are automatically covered.
>
> At larger scale, node count matters: more than 1,500 DOM nodes slows style matching, layout, and paint. The solution for long lists is virtualization — keeping only visible items in the DOM and recycling nodes as the user scrolls. Combined with DOM pooling (reusing created elements), this eliminates both allocation cost and garbage collection pressure."

---

## 19. Exercises

### Exercise 1 — Identify DOM inefficiencies

```javascript
function updateProductList(products) {
  const list = document.getElementById("product-list");
  list.innerHTML = ""; // clear list

  products.forEach((product) => {
    const item = document.createElement("div");
    item.className = "product-item";

    const name = document.createElement("h3");
    name.textContent = product.name;

    const price = document.createElement("p");
    price.textContent = "$" + product.price;

    const btn = document.createElement("button");
    btn.textContent = "Add to Cart";
    btn.addEventListener("click", () => addToCart(product.id));

    item.appendChild(name);
    item.appendChild(price);
    item.appendChild(btn);
    list.appendChild(item); // ← direct append each item
  });
}
```

Find all inefficiencies and rewrite.

<details>
<summary>Solution</summary>

```javascript
// Inefficiencies:
// 1. list.appendChild(item) inside loop — N DOM mutations
// 2. btn.addEventListener per item — N listener allocations
// 3. No template reuse — createElement called 4 times per item

// Rewritten:
const productTemplate = (() => {
  const div = document.createElement("div");
  div.className = "product-item";
  div.innerHTML = `
    <h3 class="product-name"></h3>
    <p class="product-price"></p>
    <button class="product-btn">Add to Cart</button>
  `;
  return div;
})();

function updateProductList(products) {
  const list = document.getElementById("product-list");
  const fragment = document.createDocumentFragment();

  products.forEach((product) => {
    const item = productTemplate.cloneNode(true); // reuse template
    item.dataset.productId = product.id;
    item.querySelector(".product-name").textContent = product.name;
    item.querySelector(".product-price").textContent = "$" + product.price;
    fragment.appendChild(item); // off-DOM
  });

  // Event delegation — one listener instead of N
  list.addEventListener(
    "click",
    (e) => {
      const btn = e.target.closest(".product-btn");
      if (!btn) return;
      const id = btn.closest("[data-product-id]").dataset.productId;
      addToCart(id);
    },
    { once: false },
  );

  list.replaceChildren(fragment); // single DOM mutation
}
```

</details>

---

### Exercise 2 — Build a fast list renderer

Implement `renderList(container, items)` that:

- Renders 10,000 items without freezing the UI
- Uses DocumentFragment for batch insertion
- Uses event delegation for click handling
- Each item has: name, category, price, and a "select" button

<details>
<summary>Solution</summary>

```javascript
function renderList(container, items) {
  // Template
  const template = document.createElement("div");
  template.className = "item";
  template.innerHTML = `
    <span class="item__name"></span>
    <span class="item__category"></span>
    <span class="item__price"></span>
    <button class="item__btn" data-action="select">Select</button>
  `;

  // Build fragment off-DOM
  const fragment = document.createDocumentFragment();
  items.forEach((item, index) => {
    const el = template.cloneNode(true);
    el.dataset.index = index;
    el.querySelector(".item__name").textContent = item.name;
    el.querySelector(".item__category").textContent = item.category;
    el.querySelector(".item__price").textContent = `$${item.price.toFixed(2)}`;
    fragment.appendChild(el);
  });

  // Event delegation
  container.addEventListener("click", (e) => {
    const btn = e.target.closest('[data-action="select"]');
    if (!btn) return;
    const item = btn.closest("[data-index]");
    if (!item) return;
    const selectedItem = items[parseInt(item.dataset.index, 10)];
    console.log("Selected:", selectedItem);
  });

  // Single DOM mutation
  container.replaceChildren(fragment);
}
```

</details>

---

## 🔗 Related Topics

- [`performance/03-layout-thrashing.md`](./03-layout-thrashing.md) — Layout thrashing and batching
- [`performance/02-virtualization-windowing.md`](./02-virtualization-windowing.md) — Virtualization for large lists
- [`performance/06-event-delegation.md`](./06-event-delegation.md) — Event delegation in depth
- [`rendering/01-dom-batching.md`](../rendering/01-dom-batching.md) — DOM batching patterns
- [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md) — What DOM mutations trigger

---

<div align="center">

**Next:** [`performance/02-virtualization-windowing.md`](./02-virtualization-windowing.md) →

</div>
