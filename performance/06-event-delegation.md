# 06 — Event Delegation

> **"Event delegation is one of those patterns that sounds like a minor optimization until you see a list of 10,000 items with 10,000 event listeners — and then you understand why it's architectural."**

Event delegation exploits the DOM's event propagation model to handle events from many elements with a single listener on a parent. It's not just a performance trick — it's the correct pattern for dynamic lists, it enables event handling for elements that don't exist yet, and it reduces memory usage from O(n) listeners to O(1). This document covers the event propagation model, when delegation applies, implementation patterns, and the edge cases that catch developers off guard.

---

## 📚 Table of Contents

1. [Event Propagation — The Foundation](#1-event-propagation--the-foundation)
2. [Why Delegation Works](#2-why-delegation-works)
3. [The Performance Case](#3-the-performance-case)
4. [Basic Delegation Pattern](#4-basic-delegation-pattern)
5. [`event.target` vs `event.currentTarget`](#5-eventtarget-vs-eventcurrenttarget)
6. [`closest()` — Safe Target Finding](#6-closest--safe-target-finding)
7. [Delegating Multiple Event Types](#7-delegating-multiple-event-types)
8. [Data Attributes in Delegation](#8-data-attributes-in-delegation)
9. [Delegating Keyboard Events](#9-delegating-keyboard-events)
10. [Events That Don't Bubble](#10-events-that-dont-bubble)
11. [Event Delegation for Dynamic Lists](#11-event-delegation-for-dynamic-lists)
12. [Stopping Propagation — The Tradeoffs](#12-stopping-propagation--the-tradeoffs)
13. [Building a Delegation System](#13-building-a-delegation-system)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. Event Propagation — The Foundation

When a DOM event fires, it doesn't just happen at the target element. It travels through the DOM in three phases:

```
DOM structure:
  document
    └── html
          └── body
                └── div#container
                      └── ul#list
                            ├── li.item  ← user clicks this
                            │    └── span "Item Name"
                            └── li.item

User clicks the <span> "Item Name"
```

### Phase 1 — Capture (top-down)

```
event travels DOWN from document to target:
  document → html → body → div#container → ul#list → li.item → span ← TARGET

Capture phase listeners (rare):
  element.addEventListener('click', handler, true); // third arg = capture
  element.addEventListener('click', handler, { capture: true });
```

### Phase 2 — Target

The event reaches the element that was actually clicked (`event.target`).

### Phase 3 — Bubble (bottom-up)

```
event travels UP from target to document:
  span → li.item → ul#list → div#container → body → html → document → window

Bubble phase listeners (default):
  element.addEventListener('click', handler); // bubble is default
  element.addEventListener('click', handler, false);
  element.addEventListener('click', handler, { capture: false });
```

### The Full Propagation Model

```
CLICK on <span>:

CAPTURE (down):
  document.captureListener? → fires
  html.captureListener? → fires
  body.captureListener? → fires
  #container.captureListener? → fires
  #list.captureListener? → fires
  li.captureListener? → fires
  span.captureListener? → fires (target phase actually)

BUBBLE (up):
  span.bubbleListener? → fires (target phase actually)
  li.bubbleListener? → fires  ← THIS IS THE KEY FOR DELEGATION
  #list.bubbleListener? → fires ← if listener is here, catches all li clicks
  #container.bubbleListener? → fires
  body.bubbleListener? → fires
  html.bubbleListener? → fires
  document.bubbleListener? → fires
  window.bubbleListener? → fires
```

---

## 2. Why Delegation Works

Because events bubble up, an ancestor can catch events from all descendants:

```javascript
// Without delegation: 1000 listeners for 1000 items
items.forEach((item) => {
  item.addEventListener("click", handleItemClick); // N listeners
});

// With delegation: 1 listener catches all clicks on any item
list.addEventListener("click", (event) => {
  const item = event.target.closest(".item");
  if (item) handleItemClick(item);
}); // 1 listener
```

When any `.item` descendant is clicked, the event bubbles up to `#list` where the single listener catches it. The listener then uses `event.target` to determine which item was clicked.

---

## 3. The Performance Case

### Memory Cost Comparison

```javascript
// 10,000 list items without delegation:
items.forEach((item) => {
  item.addEventListener("click", () => {
    // Each callback is a closure capturing `item`
    doSomethingWith(item);
  });
});

// Memory breakdown (per listener):
//   Function object: ~100-300 bytes
//   Closure environment: ~100-500 bytes (captures item reference)
//   Browser event listener registration: ~50-100 bytes
// Total per listener: ~300-900 bytes

// For 10,000 items: 10,000 × 600 bytes ≈ 6MB just for event listeners
// Plus: 10,000 closure allocations → GC pressure

// With delegation:
list.addEventListener("click", handler); // ONE listener, ~300 bytes
// Savings: ~5.997MB
```

### Event Registration Cost

```javascript
// Benchmarking event listener attachment
const COUNT = 10_000;
const items = Array.from({ length: COUNT }, (_, i) => {
  const li = document.createElement("li");
  li.className = "item";
  li.dataset.id = String(i);
  return li;
});

// Method 1: individual listeners
console.time("individual");
items.forEach((item) => {
  item.addEventListener("click", handler);
});
console.timeEnd("individual");
// Result: ~15-25ms for 10,000 listeners

// Method 2: delegation
console.time("delegation");
list.addEventListener("click", delegateHandler);
console.timeEnd("delegation");
// Result: ~0.05ms
// ~300-500× faster to register
```

---

## 4. Basic Delegation Pattern

```javascript
// Structure:
// <ul id="list">
//   <li class="item" data-id="1">
//     <span class="item__name">Widget</span>
//     <button class="item__btn">Delete</button>
//   </li>
//   ...
// </ul>

const list = document.getElementById("list");

list.addEventListener("click", (event) => {
  // event.target = the element that was actually clicked
  // Could be: li.item, span.item__name, button.item__btn
  // (depends on exactly which pixel the user clicked)

  // Find the closest ancestor (or self) matching our selector
  const item = event.target.closest(".item");

  // If click was outside any .item (e.g., on list padding): item is null
  if (!item) return;

  const id = item.dataset.id;
  console.log("Clicked item:", id);
});
```

### Why `closest()` Instead of Direct `target` Check

```html
<!-- User clicks the <strong> inside the <li> -->
<ul id="list">
  <li class="item" data-id="1">
    <strong>Widget <em>Pro</em></strong>
    <button>Delete</button>
  </li>
</ul>
```

```javascript
list.addEventListener("click", (event) => {
  // ❌ Direct target check fails when nested elements are clicked
  if (event.target.className === "item") {
    // If user clicks <strong> or <em>: event.target is NOT .item
    // This condition fails — click is silently ignored
  }

  // ✅ closest() walks up the DOM to find .item
  const item = event.target.closest(".item");
  // If user clicks <strong>: target is <strong>
  // closest('.item') walks up: strong → li.item ← found!
  // Works regardless of how deeply nested the click target is
});
```

---

## 5. `event.target` vs `event.currentTarget`

This distinction is central to delegation:

```javascript
// HTML:
// <ul id="list">
//   <li class="item"><span>Text</span></li>
// </ul>

document.getElementById("list").addEventListener("click", (event) => {
  console.log(event.target); // the element actually clicked
  console.log(event.currentTarget); // the element the listener is on

  // If user clicks the <span>:
  //   event.target:        <span>
  //   event.currentTarget: <ul#list>

  // If user clicks the <li>:
  //   event.target:        <li class="item">
  //   event.currentTarget: <ul#list>

  // If user clicks the <ul> itself (between items):
  //   event.target:        <ul#list>
  //   event.currentTarget: <ul#list> ← same in this case
});
```

### When They Matter

```javascript
list.addEventListener("click", (event) => {
  // event.currentTarget: ALWAYS the list (#list)
  // Use for: comparing against the container

  // event.target: whatever was clicked
  // Use for: finding which item was interacted with

  // Safe delegation pattern:
  const item = event.target.closest(".item");
  if (!item || !list.contains(item)) return;
  // The contains() check ensures item is inside our list
  // (guards against edge cases where closest() finds something outside)
});
```

---

## 6. `closest()` — Safe Target Finding

`Element.closest(selector)` traverses up from `event.target` to find the nearest ancestor (or the element itself) matching the selector. Returns `null` if none found.

```javascript
// Safe pattern for delegation:
container.addEventListener("click", (event) => {
  // Step 1: Find the logical target (the item, not the sub-element)
  const item = event.target.closest("[data-item-id]");

  // Step 2: Guard against clicks outside any item
  if (!item) return;

  // Step 3: Guard against items outside our container
  // (closest() traverses the entire DOM tree, could escape container)
  if (!container.contains(item)) return;

  // Step 4: Extract data and handle
  const id = item.dataset.itemId;
  handleItemAction(id, event);
});
```

### Combining `closest()` for Different Actions

```javascript
// Delegation for multiple action types in one list
list.addEventListener("click", (event) => {
  // Check for delete button
  const deleteBtn = event.target.closest('[data-action="delete"]');
  if (deleteBtn) {
    const id = deleteBtn.closest("[data-id]").dataset.id;
    deleteItem(id);
    return;
  }

  // Check for edit button
  const editBtn = event.target.closest('[data-action="edit"]');
  if (editBtn) {
    const id = editBtn.closest("[data-id]").dataset.id;
    editItem(id);
    return;
  }

  // Check for row selection
  const row = event.target.closest("[data-id]");
  if (row) {
    selectItem(row.dataset.id);
  }
});
```

---

## 7. Delegating Multiple Event Types

One container can handle multiple event types through multiple listeners, all delegated:

```javascript
const table = document.getElementById("data-table");

// Click delegation
table.addEventListener("click", (event) => {
  const cell = event.target.closest("td[data-col]");
  if (cell) selectCell(cell.dataset.col, cell.closest("tr").dataset.row);
});

// Double-click delegation
table.addEventListener("dblclick", (event) => {
  const cell = event.target.closest("td[data-col]");
  if (cell) editCell(cell);
});

// Right-click delegation
table.addEventListener("contextmenu", (event) => {
  const row = event.target.closest("tr[data-row]");
  if (row) {
    event.preventDefault();
    showContextMenu(event.clientX, event.clientY, row.dataset.row);
  }
});

// Keyboard delegation (on table for focus management)
table.addEventListener("keydown", (event) => {
  const cell = document.activeElement.closest("td[data-col]");
  if (!cell) return;

  if (event.key === "Enter") editCell(cell);
  if (event.key === "Delete") clearCell(cell);
  if (event.key === "ArrowRight") moveFocus(cell, "right");
  if (event.key === "ArrowLeft") moveFocus(cell, "left");
});
```

---

## 8. Data Attributes in Delegation

Data attributes on the item elements are the clean way to pass context to delegated handlers — no closures needed.

```html
<!-- Action and context encoded in the element itself -->
<ul id="product-list">
  <li data-id="42" data-category="electronics" data-in-stock="true">
    <span class="name">Widget Pro</span>
    <span class="price">$29.99</span>
    <button data-action="add-to-cart">Add to Cart</button>
    <button data-action="wishlist">♡ Wishlist</button>
    <button data-action="share">Share</button>
  </li>
  <!-- ... more items ... -->
</ul>
```

```javascript
document.getElementById("product-list").addEventListener("click", (event) => {
  const button = event.target.closest("button[data-action]");
  if (!button) return;

  const item = button.closest("[data-id]");
  if (!item) return;

  const action = button.dataset.action;
  const id = item.dataset.id;
  const category = item.dataset.category;
  const inStock = item.dataset.inStock === "true";

  switch (action) {
    case "add-to-cart":
      if (!inStock) return;
      cart.add(id, category);
      break;
    case "wishlist":
      wishlist.toggle(id);
      break;
    case "share":
      share(id);
      break;
  }
});
```

---

## 9. Delegating Keyboard Events

```javascript
// Keyboard navigation with delegation
// <ul role="listbox" tabindex="0" id="dropdown">
//   <li role="option" data-value="a" tabindex="-1">Option A</li>
//   <li role="option" data-value="b" tabindex="-1">Option B</li>
// </ul>

const dropdown = document.getElementById("dropdown");
let currentIndex = -1;

dropdown.addEventListener("keydown", (event) => {
  const options = [...dropdown.querySelectorAll('[role="option"]')];

  switch (event.key) {
    case "ArrowDown":
      event.preventDefault();
      currentIndex = Math.min(currentIndex + 1, options.length - 1);
      options[currentIndex].focus();
      break;

    case "ArrowUp":
      event.preventDefault();
      currentIndex = Math.max(currentIndex - 1, 0);
      options[currentIndex].focus();
      break;

    case "Enter":
    case " ":
      event.preventDefault();
      const focused = document.activeElement.closest('[role="option"]');
      if (focused && dropdown.contains(focused)) {
        selectOption(focused.dataset.value);
      }
      break;

    case "Escape":
      dropdown.focus(); // return focus to container
      closeDropdown();
      break;

    case "Home":
      event.preventDefault();
      currentIndex = 0;
      options[0]?.focus();
      break;

    case "End":
      event.preventDefault();
      currentIndex = options.length - 1;
      options[currentIndex]?.focus();
      break;
  }
});
```

---

## 10. Events That Don't Bubble

Not all events bubble — delegation only works for events that do.

### Events That Bubble ✅

```
click, dblclick, mousedown, mouseup, mousemove
keydown, keyup, keypress
input, change
submit, reset
focus (via focusin — focus itself doesn't bubble!)
blur (via focusout — blur itself doesn't bubble!)
touchstart, touchend, touchmove
drag, dragstart, dragend, dragover, dragenter, dragleave, drop
wheel, scroll (scroll doesn't bubble on most elements)
contextmenu
pointerdown, pointerup, pointermove, pointerover, pointerout
```

### Events That DON'T Bubble ❌

```
focus, blur         → use focusin, focusout instead
mouseenter, mouseleave → use mouseover, mouseout instead
load, unload, error (on elements)
resize
scroll (on window, but not elements — actually does on document)
```

### Workaround for Non-Bubbling Events

```javascript
// ❌ focus doesn't bubble — can't delegate
list.addEventListener("focus", (e) => {
  // This only fires when #list itself is focused, NOT its children
  const input = e.target.closest("input");
  if (input) highlightRow(input);
});

// ✅ focusin bubbles — use this for delegation
list.addEventListener("focusin", (e) => {
  const input = e.target.closest("input");
  if (input) highlightRow(input);
});

// ✅ Alternative: capture phase (non-bubbling events DO fire in capture)
list.addEventListener(
  "focus",
  (e) => {
    const input = e.target.closest("input");
    if (input) highlightRow(input);
  },
  { capture: true },
); // capture: true intercepts during descend phase
```

---

## 11. Event Delegation for Dynamic Lists

Delegation truly shines with dynamic content — items added after initial render are automatically handled.

```javascript
class DynamicList {
  #container;
  #items = [];

  constructor(container) {
    this.#container = container;

    // One listener handles ALL items — current AND future
    this.#container.addEventListener("click", this.#handleClick.bind(this));
  }

  addItem(item) {
    const el = document.createElement("li");
    el.className = "item";
    el.dataset.id = item.id;
    el.innerHTML = `
      <span class="item__label">${item.label}</span>
      <button data-action="edit">Edit</button>
      <button data-action="delete">Delete</button>
    `;
    this.#container.appendChild(el);
    this.#items.push(item);
    // No event listener attached — delegation handles it automatically
  }

  removeItem(id) {
    const el = this.#container.querySelector(`[data-id="${id}"]`);
    el?.remove();
    // No listener to clean up — nothing was attached to the element
    this.#items = this.#items.filter((item) => item.id !== id);
  }

  #handleClick(event) {
    const action = event.target.closest("[data-action]")?.dataset.action;
    const item = event.target.closest("[data-id]");
    if (!action || !item) return;

    const id = item.dataset.id;
    if (action === "edit") this.#editItem(id);
    if (action === "delete") this.removeItem(id);
  }

  destroy() {
    // Only ONE listener to remove — clean and simple
    this.#container.removeEventListener("click", this.#handleClick);
  }
}
```

---

## 12. Stopping Propagation — The Tradeoffs

`stopPropagation()` stops the event from bubbling further. It's sometimes necessary but can break delegation.

### When `stopPropagation` Breaks Delegation

```javascript
// ❌ Inner button stops propagation — delegation on container won't fire
button.addEventListener("click", (event) => {
  event.stopPropagation(); // stops bubble
  handleButtonClick();
});

container.addEventListener("click", (event) => {
  // If button is inside container and stopPropagation was called:
  // This listener NEVER FIRES when button is clicked
  // Delegation is broken
});
```

### Alternatives to `stopPropagation`

```javascript
// ❌ Using stopPropagation to prevent container click
button.addEventListener("click", (event) => {
  event.stopPropagation();
  doButtonThing();
});
container.addEventListener("click", () => {
  selectRow(); // Don't want this to fire when button is clicked
});

// ✅ Better: check target in container handler
container.addEventListener("click", (event) => {
  // If click was on or inside a button, don't select row
  if (event.target.closest("button")) return;
  selectRow();
});

// ✅ Or: use data attributes to signal intent
container.addEventListener("click", (event) => {
  if (event.target.closest("[data-no-select]")) return;
  selectRow();
});
// HTML: <button data-no-select data-action="edit">Edit</button>
```

### `preventDefault` vs `stopPropagation`

```javascript
// preventDefault: stops the browser's default action
// (does NOT stop propagation)
link.addEventListener("click", (event) => {
  event.preventDefault(); // browser won't navigate to href
  handleCustomNavigation(); // but event still bubbles
});

// stopPropagation: stops event bubbling
// (does NOT prevent default)
item.addEventListener("click", (event) => {
  event.stopPropagation(); // event won't reach parent handlers
  // but default action (e.g., following a link) still happens
});

// stopImmediatePropagation: stops all listeners on current element + bubbling
element.addEventListener("click", handler1);
element.addEventListener("click", (event) => {
  event.stopImmediatePropagation();
  // handler1 and any parent handlers won't fire
});
```

---

## 13. Building a Delegation System

A reusable delegation utility that handles the common pattern:

```javascript
/**
 * Attach a delegated event listener to a container.
 * @param {Element} container - The container element to listen on
 * @param {string} eventType - Event type (click, keydown, etc.)
 * @param {string} selector - CSS selector for target elements
 * @param {Function} handler - Called with (event, matchedElement)
 * @returns {Function} - Remove the listener when called
 */
function delegate(container, eventType, selector, handler) {
  function listener(event) {
    const target = event.target.closest(selector);
    if (target && container.contains(target)) {
      handler.call(target, event, target);
    }
  }

  container.addEventListener(eventType, listener);

  // Return cleanup function
  return () => container.removeEventListener(eventType, listener);
}

// Usage
const stopListening = delegate(
  document.getElementById("product-list"),
  "click",
  "[data-action]",
  function (event, element) {
    // `this` and `element` both = the matched [data-action] element
    const action = element.dataset.action;
    const itemId = element.closest("[data-id]")?.dataset.id;
    handleAction(action, itemId);
  },
);

// Later: remove listener
stopListening();
```

### Multi-Event Delegation Helper

```javascript
class EventDelegator {
  #container;
  #listeners = [];

  constructor(container) {
    this.#container = container;
  }

  on(eventType, selector, handler) {
    const unsubscribe = delegate(this.#container, eventType, selector, handler);
    this.#listeners.push(unsubscribe);
    return this; // chainable
  }

  destroy() {
    this.#listeners.forEach((fn) => fn());
    this.#listeners = [];
  }
}

// Usage
const delegator = new EventDelegator(document.getElementById("app"));

delegator
  .on("click", '[data-action="delete"]', (e, el) => deleteItem(el.dataset.id))
  .on("click", '[data-action="edit"]', (e, el) => editItem(el.dataset.id))
  .on("keydown", '[role="option"]', handleOptionKeydown)
  .on("focusin", ".input-field", highlightParentRow);

// Clean up:
delegator.destroy();
```

---

## 14. Good Practices

### ✅ Delegate at the lowest useful ancestor

```javascript
// ❌ Too high: document catches everything on the entire page
document.addEventListener("click", handleProductClick);

// ✅ Scoped to the relevant container
document
  .getElementById("product-grid")
  .addEventListener("click", handleProductClick);
// Clicks outside product-grid don't even reach this listener
```

### ✅ Use `data-action` for intent clarity

```html
<!-- ✅ Action encoded in DOM — clear intent, easy to delegate -->
<button data-action="add-to-cart" data-product-id="42">Add to Cart</button>
<button data-action="save-draft">Save Draft</button>
<button data-action="publish">Publish</button>
```

```javascript
container.addEventListener("click", (event) => {
  const { action } = event.target.closest("[data-action]")?.dataset ?? {};
  if (!action) return;
  actions[action]?.(event);
});
```

### ✅ Always guard with `closest()` and `null` check

```javascript
// ✅ Always defensive
container.addEventListener("click", (event) => {
  const item = event.target.closest(".item");
  if (!item) return; // clicks outside items: no-op
  // safe to proceed
});
```

### ✅ Use `focusin`/`focusout` instead of `focus`/`blur` for delegation

```javascript
// ✅ focusin and focusout bubble — delegate them
form.addEventListener("focusin", (e) =>
  highlightField(e.target.closest(".field")),
);
form.addEventListener("focusout", (e) =>
  validateField(e.target.closest(".field")),
);
```

---

## 15. Bad Practices

### ❌ Calling `stopPropagation` globally in child components

```javascript
// ❌ Common React/Vue anti-pattern:
// Child component stops all click propagation
element.addEventListener("click", (e) => {
  e.stopPropagation(); // "prevent bubbling to parent"
  doThing();
});

// Breaks: any parent-level delegation
// Breaks: analytics event tracking at root
// Breaks: click-outside detection patterns
// Fix: handle the specific case in the parent handler instead
```

### ❌ Querying children inside a delegated handler

```javascript
// ❌ Runs querySelectorAll on every click — expensive
container.addEventListener("click", (event) => {
  const allItems = container.querySelectorAll(".item"); // re-query on every click
  const clickedItem = allItems.find((item) => item.contains(event.target));
  // O(n) query + O(n) find on every click
});

// ✅ Use closest()
container.addEventListener("click", (event) => {
  const item = event.target.closest(".item"); // O(depth) — fast
});
```

### ❌ Using delegation for one-time or rare interactions

```javascript
// ❌ Delegation overhead not worth it for a single static element
document.addEventListener("click", (event) => {
  if (event.target.closest("#close-modal-btn")) {
    closeModal();
  }
});

// ✅ Just attach directly to the element
document
  .getElementById("close-modal-btn")
  .addEventListener("click", closeModal);
```

---

## 16. Common Mistakes

### Mistake 1 — Forgetting `closest()` returns null outside the container

```javascript
// ❌ closest() traverses the ENTIRE DOM tree — can escape container
const item = event.target.closest(".item");
// What if event.target is inside a modal that also has .item?
// closest() finds the modal's .item, not the list's .item

// ✅ Verify it's within our container
const item = event.target.closest(".item");
if (!item || !container.contains(item)) return;
```

### Mistake 2 — Expecting delegation to work for non-bubbling events

```javascript
// ❌ mouseenter doesn't bubble — handler never fires for children
container.addEventListener("mouseenter", (event) => {
  const item = event.target.closest(".item"); // target is always container
  // This never finds .item because mouseenter fires directly on container
});

// ✅ Use mouseover (which does bubble)
container.addEventListener("mouseover", (event) => {
  const item = event.target.closest(".item");
  if (item && !item.matches(":hover ~ *")) {
    // first entry only
    highlightItem(item);
  }
});
```

### Mistake 3 — Attaching delegation listener inside a loop

```javascript
// ❌ Creates N listeners — defeats the purpose
rows.forEach((row) => {
  tableBody.addEventListener("click", (event) => {
    // ...
  });
});
// Now there are N listeners on tableBody!

// ✅ Attach once, outside the loop
tableBody.addEventListener("click", (event) => {
  const row = event.target.closest("tr[data-id]");
  if (row) handleRowClick(row);
});
```

### Mistake 4 — Not accounting for SVG and custom elements

```javascript
// SVG elements have different className API (SVGAnimatedString, not string)
// ❌ className.includes doesn't work on SVG
event.target.className.includes("icon"); // throws or returns wrong result

// ✅ Use matches() or closest() instead
event.target.closest(".icon");
event.target.matches(".icon");
event.target.classList.contains("icon"); // classList works on SVG too
```

---

## 17. Interview-Level Explanation

> **"What is event delegation? Why use it? What are the tradeoffs?"**

**Strong answer:**

> "Event delegation is a pattern that attaches a single event listener to a parent element instead of individual listeners to each child. It exploits the DOM's event bubbling model: when any element is clicked, the event bubbles up through all its ancestors, so a listener on the parent catches events from any descendant.
>
> The main benefits are memory efficiency and dynamic content support. Without delegation, a list of 10,000 items would require 10,000 event listeners — each is an object in memory, each has a closure potentially capturing item data. With delegation, one listener handles all 10,000 items. More importantly, items added dynamically after the initial render are automatically handled — no code to attach or remove listeners on add/remove.
>
> The implementation pattern uses `event.target.closest(selector)` rather than checking `event.target` directly. This is critical because the actual clicked element might be a child — a `<strong>` inside a `<li>` for example. `closest()` walks up the DOM from the clicked element to find the nearest ancestor matching your selector.
>
> The key limitation is that not all events bubble. `focus` and `blur` don't bubble, so you need `focusin` and `focusout` instead. `mouseenter` and `mouseleave` don't bubble — use `mouseover` and `mouseout`. And delegation can be broken by `stopPropagation()` in child elements, which is why stopping propagation should generally be avoided in favor of checking the event target in the parent handler.
>
> When NOT to use delegation: for a single static element, direct attachment is cleaner. For elements that need fine-grained control (like `mouseenter` for hover effects), direct listeners or CSS `:hover` are better. Delegation shines for dynamic lists, table rows, repeated UI elements, and any time N > 50 or so."

---

## 18. Exercises

### Exercise 1 — Refactor to use delegation

```javascript
// ❌ Current code: individual listeners
function initProductGrid(products) {
  const grid = document.getElementById("grid");
  grid.innerHTML = "";

  products.forEach((product) => {
    const card = document.createElement("div");
    card.className = "card";
    card.innerHTML = `
      <h3>${product.name}</h3>
      <p>$${product.price}</p>
      <button class="btn-cart">Add to Cart</button>
      <button class="btn-wishlist">♡</button>
    `;

    card.querySelector(".btn-cart").addEventListener("click", () => {
      cart.add(product.id);
    });
    card.querySelector(".btn-wishlist").addEventListener("click", () => {
      wishlist.toggle(product.id);
    });

    grid.appendChild(card);
  });
}
```

Refactor to use delegation with data attributes.

<details>
<summary>Solution</summary>

```javascript
function initProductGrid(products) {
  const grid = document.getElementById("grid");

  // Build HTML — store IDs in data attributes
  const fragment = document.createDocumentFragment();
  products.forEach((product) => {
    const card = document.createElement("div");
    card.className = "card";
    card.dataset.productId = product.id;
    card.innerHTML = `
      <h3>${product.name}</h3>
      <p>$${product.price}</p>
      <button data-action="add-to-cart">Add to Cart</button>
      <button data-action="wishlist">♡</button>
    `;
    fragment.appendChild(card);
  });
  grid.replaceChildren(fragment);

  // ONE delegated listener for all cards, current and future
  grid.addEventListener("click", (event) => {
    const button = event.target.closest("[data-action]");
    if (!button) return;

    const card = button.closest("[data-product-id]");
    if (!card) return;

    const id = card.dataset.productId;
    const action = button.dataset.action;

    if (action === "add-to-cart") cart.add(id);
    if (action === "wishlist") wishlist.toggle(id);
  });
}
```

</details>

---

### Exercise 2 — Handle keyboard navigation with delegation

Implement keyboard navigation for a custom `<select>` replacement:

```html
<div id="listbox" role="listbox" tabindex="0" aria-label="Choose country">
  <div role="option" tabindex="-1" data-value="us">United States</div>
  <div role="option" tabindex="-1" data-value="uk">United Kingdom</div>
  <div role="option" tabindex="-1" data-value="ca">Canada</div>
</div>
```

Implement: ArrowDown/Up to move focus, Enter/Space to select, Home/End for first/last.

<details>
<summary>Solution</summary>

```javascript
const listbox = document.getElementById("listbox");

listbox.addEventListener("keydown", (event) => {
  const options = [...listbox.querySelectorAll('[role="option"]')];
  const focused = document.activeElement;
  const currentIdx = options.indexOf(focused);

  let nextIdx = currentIdx;

  switch (event.key) {
    case "ArrowDown":
      event.preventDefault();
      nextIdx = currentIdx < options.length - 1 ? currentIdx + 1 : 0;
      break;
    case "ArrowUp":
      event.preventDefault();
      nextIdx = currentIdx > 0 ? currentIdx - 1 : options.length - 1;
      break;
    case "Home":
      event.preventDefault();
      nextIdx = 0;
      break;
    case "End":
      event.preventDefault();
      nextIdx = options.length - 1;
      break;
    case "Enter":
    case " ":
      event.preventDefault();
      if (focused && listbox.contains(focused)) {
        selectOption(focused.dataset.value);
      }
      return;
    default:
      return;
  }

  options[nextIdx]?.focus();
});

function selectOption(value) {
  const option = listbox.querySelector(`[data-value="${value}"]`);
  if (!option) return;

  // Update aria state
  listbox.querySelectorAll('[role="option"]').forEach((opt) => {
    opt.setAttribute("aria-selected", opt === option ? "true" : "false");
  });

  listbox.setAttribute("aria-activedescendant", option.id);
  console.log("Selected:", value);
}
```

</details>

---

## 🔗 Related Topics

- [`performance/01-dom-optimization.md`](./01-dom-optimization.md) — DOM fundamentals including listener cost
- [`browser-internals/02-dom-tree-creation.md`](../browser-internals/02-dom-tree-creation.md) — DOM structure and event model
- [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md) — Observer pattern vs delegation
- [`performance/02-virtualization-windowing.md`](./02-virtualization-windowing.md) — Delegation in virtualized lists

---

<div align="center">

**Next:** [`performance/07-memoization.md`](./07-memoization.md) →

</div>
