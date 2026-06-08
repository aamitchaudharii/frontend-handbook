# 02 — Virtual DOM

> **"The virtual DOM is not faster than direct DOM manipulation. It's faster than naive direct DOM manipulation — and more importantly, it's faster than asking the developer to correctly and efficiently batch DOM updates by hand for every change in a complex UI. The virtual DOM is a performance floor, not a ceiling."**

The virtual DOM (VDOM) is an in-memory representation of the real DOM. Instead of updating the browser DOM directly on every state change, a library like React maintains a lightweight JavaScript object tree that mirrors the DOM structure. When state changes, a new virtual tree is generated and diffed against the previous one. Only the minimal set of real DOM mutations needed to bring the DOM up to date with the new tree are applied. This document covers how virtual DOM diffing works, React's reconciliation algorithm, fiber, concurrent mode, and when the VDOM model has limitations.

---

## 📚 Table of Contents

1. [What Virtual DOM Is](#1-what-virtual-dom-is)
2. [The Reconciliation Algorithm](#2-the-reconciliation-algorithm)
3. [The Diffing Heuristics](#3-the-diffing-heuristics)
4. [Keys — The Diffing Identity Signal](#4-keys--the-diffing-identity-signal)
5. [React Fiber — The Architecture](#5-react-fiber--the-architecture)
6. [Concurrent Mode and Scheduling](#6-concurrent-mode-and-scheduling)
7. [Building a Minimal VDOM](#7-building-a-minimal-vdom)
8. [React Server Components — Beyond VDOM](#8-react-server-components--beyond-vdom)
9. [Alternatives to Virtual DOM](#9-alternatives-to-virtual-dom)
10. [Good Practices](#10-good-practices)
11. [Bad Practices](#11-bad-practices)
12. [Common Mistakes](#12-common-mistakes)
13. [Interview-Level Explanation](#13-interview-level-explanation)
14. [Exercises](#14-exercises)

---

## 1. What Virtual DOM Is

### The Problem VDOM Solves

```javascript
// Without VDOM: every state change requires manual DOM surgery
function updateUI(state) {
  // Developer must figure out exactly what changed
  if (state.count !== prevState.count) {
    document.getElementById("count").textContent = state.count;
  }
  if (state.items.length !== prevState.items.length) {
    // Diff items manually — which were added/removed/reordered?
    // This is complex, error-prone, and must be written for every component
  }
  // ...
}

// With VDOM: describe what the UI should look like, let the library diff
function render(state) {
  return (
    <div>
      <p>Count: {state.count}</p>
      <ul>
        {state.items.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
  // Library handles: what changed? what DOM operations are needed?
}
```

### Virtual DOM as JavaScript Objects

```javascript
// JSX compiles to React.createElement calls:
const element = (
  <div className="card">
    <h2>Hello</h2>
    <p>World</p>
  </div>
);

// Which is:
const element = React.createElement(
  "div",
  { className: "card" },
  React.createElement("h2", null, "Hello"),
  React.createElement("p", null, "World"),
);

// Which produces:
const element = {
  type: "div",
  props: {
    className: "card",
    children: [
      { type: "h2", props: { children: "Hello" }, key: null, ref: null },
      { type: "p", props: { children: "World" }, key: null, ref: null },
    ],
  },
  key: null,
  ref: null,
  $$typeof: Symbol(react.element), // for security
};

// This lightweight object IS the virtual DOM node
// Creating it: ~100ns (trivial JavaScript object creation)
// Creating equivalent real DOM node: ~1000ns+ (C++ allocation, style computation)
```

### The Full VDOM Lifecycle

```
STATE CHANGE:
  1. setState() called
  2. React schedules a re-render (batched)
  3. Component function called: returns new JSX tree
  4. JSX compiled to new virtual DOM tree (JavaScript objects)

RECONCILIATION:
  5. React diffs new virtual tree against previous virtual tree
  6. Identifies minimum set of changes needed
  7. Creates a "list of effects" (DOM mutations to apply)

COMMIT PHASE:
  8. React applies effects to real DOM
  9. DOM is now consistent with new state
  10. useEffect hooks run (after DOM updates)
```

---

## 2. The Reconciliation Algorithm

React's reconciliation algorithm determines the minimum set of DOM operations needed to transform the old tree into the new tree.

### Two-Phase Rendering

```
RENDER PHASE (pure, interruptible):
  - Calls component functions to produce virtual DOM
  - Diffs new virtual DOM against previous
  - Builds a list of side effects
  - Can be interrupted and restarted (concurrent mode)
  - No real DOM mutations

COMMIT PHASE (synchronous, non-interruptible):
  - Applies all side effects to real DOM
  - Calls lifecycle methods (useLayoutEffect, componentDidMount)
  - Cannot be interrupted — DOM must reach a consistent state

AFTER COMMIT:
  - Calls useEffect hooks (asynchronous)
  - Schedules any pending work
```

### The Diffing Process

```
Old virtual tree:     New virtual tree:
  <div>                 <div>
    <h1>Hello</h1>        <h1>Hello</h1>   ← same type, same children: skip
    <p>Old text</p>       <p>New text</p>  ← same type, different children: update text
    <Button />            <Link />         ← different type: remove Button, add Link
  </div>               </div>

React's decisions:
  h1: same type + same props → nothing to do
  p:  same type + different children → update text node
  Button vs Link: different types → unmount Button, mount Link

DOM operations:
  1. Set p's text content to "New text"
  2. Remove Button's DOM node
  3. Insert Link's DOM node
  Minimum operations: 3
```

---

## 3. The Diffing Heuristics

The naive optimal tree diff algorithm is O(n³). React's diff uses O(n) heuristics:

### Heuristic 1 — Different Element Types Produce Different Trees

```jsx
// ❌ Bad: changes wrapper type → full subtree remount
// Before:
<div>
  <Counter />
</div>

// After:
<span>  {/* type changed from div to span */}
  <Counter />
</span>

// React: div ≠ span → destroy entire subtree → mount entirely new subtree
// Counter loses all state — it was unmounted and remounted
```

```jsx
// ✅ Good: keep the same wrapper type when content changes
// Before:
<div className="card">
  <Counter />
</div>

// After:
<div className="card selected"> {/* only className changed */}
  <Counter />
</div>

// React: div === div → update className → Counter keeps state
```

### Heuristic 2 — Component Type Identity

```jsx
// If the component type changes, React unmounts and remounts
// Even if the rendered output would be identical

// Before:
<FancyInput />  // → renders <input className="fancy">

// After:
<BasicInput />  // → also renders <input className="fancy">

// React: FancyInput !== BasicInput → unmount FancyInput (loses state)
//        → mount BasicInput (starts fresh)
// Even though the DOM output is identical!
```

### Heuristic 3 — List Items Need Keys

```jsx
// Without keys: React can't tell which items are the same
// Before: [A, B, C]
// After:  [B, C, D]  (A removed, D added at end)

// React without keys: compares by position
// Position 0: A vs B → update
// Position 1: B vs C → update
// Position 2: C vs D → update
// 3 DOM updates

// React with keys:
// Key B: B→B (same), no update
// Key C: C→C (same), no update
// Key A: removed, 1 DOM removal
// Key D: added, 1 DOM insertion
// 2 DOM operations (better)
```

---

## 4. Keys — The Diffing Identity Signal

Keys tell React which virtual DOM nodes correspond to which real DOM nodes across renders.

### Stable Keys for Lists

```jsx
// ✅ Stable, unique keys from data
function ProductList({ products }) {
  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          {" "}
          {/* stable key from data */}
          <ProductCard product={product} />
        </li>
      ))}
    </ul>
  );
}
// Reordering products: React matches by key, moves DOM nodes
// No remounts, no lost state

// ❌ Array index as key — breaks when list reorders
function ProductList({ products }) {
  return (
    <ul>
      {products.map((product, index) => (
        <li key={index}>
          {" "}
          {/* ❌ index changes on reorder */}
          <ProductCard product={product} />
        </li>
      ))}
    </ul>
  );
}
// Inserting at beginning: ALL keys shift → all items remount
// Sorting: all keys change → all items remount
```

### When Index Key Is Acceptable

```jsx
// ✅ Index key is acceptable only when:
// 1. The list never reorders
// 2. The list never inserts in the middle
// 3. Items have no local state (stateless display only)

// Acceptable: a list of simple display-only rows that never reorders
function ReadonlyLog({ messages }) {
  return messages.map((msg, i) => (
    <span key={i}>{msg}</span> // ok: no reorder, no local state, display only
  ));
}
```

### Keys for Forced Remounts

```jsx
// Keys can force remount — useful for resetting component state
function Editor({ documentId }) {
  return (
    // When documentId changes: Editor remounts from scratch (resets all state)
    <RichTextEditor key={documentId} />
  );
}

// Without key={documentId}: Editor keeps stale state from previous document
// With key={documentId}: Editor remounts when documentId changes → clean state
```

---

## 5. React Fiber — The Architecture

React Fiber is the complete rewrite of React's reconciliation engine (React 16+). It enables incremental rendering and priority-based scheduling.

### The Problem with the Old Stack Reconciler

```
Old stack reconciler:
  Reconciliation was synchronous and non-interruptible.
  A large component tree update would block the main thread
  until the entire tree was reconciled.

  500 components to reconcile × 0.5ms each = 250ms
  Main thread blocked for 250ms:
  - Animations freeze
  - Input is unresponsive
  - User perceives jank
```

### Fiber's Solution — Work as Units

```
Fiber architecture:
  Each component = one "fiber" unit of work
  React processes fibers incrementally
  Between fibers: check if higher-priority work is waiting
  If so: pause current work, do higher-priority work, resume

  Result: animations and input handling are never starved
  Long reconciliations are interleaved with rendering frames
```

### The Fiber Data Structure

```javascript
// A Fiber is a JavaScript object representing a unit of work
const fiber = {
  // Identity
  type:      Button,          // component type or HTML tag
  key:       'button-1',     // key prop

  // State
  stateNode: domNode,         // real DOM node or component instance
  pendingProps: newProps,     // props for this render
  memoizedProps: prevProps,   // props from last render
  memoizedState: currentState, // state from last render

  // Tree structure (linked list — not nested objects)
  return:  parentFiber,       // parent fiber
  child:   firstChildFiber,   // first child
  sibling: nextSiblingFiber,  // next sibling (not in an array)

  // Effects
  effectTag: Update,          // type of DOM operation needed (Insert/Update/Delete)
  nextEffect: nextEffectFiber, // next fiber with a side effect (linked list)

  // Work priority
  lanes: ...,                 // which priority lanes this work is for

  // Double-buffering
  alternate: workInProgressFiber, // the "work in progress" copy
};
```

### Double Buffering — Two Fiber Trees

```
React maintains two fiber trees:
  "current" tree:  the currently rendered version (shown to user)
  "workInProgress" tree: the next version being prepared

Work cycle:
  1. Start with current tree (what user sees)
  2. Clone into workInProgress tree
  3. Process changes in workInProgress (interruptible)
  4. When done: swap workInProgress → current
  5. Apply real DOM mutations
  6. Old current tree becomes next workInProgress (reuse memory)
```

---

## 6. Concurrent Mode and Scheduling

Concurrent Mode enables React to prepare multiple versions of the UI simultaneously and show them based on priority.

### Priority Lanes

```javascript
// React 18 uses a "lanes" model for priority
// (simplified representation)

const SyncLane = 0b0000000000000000000000000000001; // highest
const InputContinuousLane = 0b0000000000000000000000000000100;
const DefaultLane = 0b0000000000000000000000000010000;
const TransitionLane = 0b0000000000000000000000001000000;
const IdleLane = 0b0100000000000000000000000000000; // lowest

// User input: SyncLane → processed immediately
// Network data: DefaultLane → processed soon
// Transitions: TransitionLane → can be interrupted by higher priority
// Prefetch: IdleLane → only when nothing else is happening
```

### `startTransition` — Deferring Low-Priority Updates

```jsx
import { useState, startTransition } from "react";

function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  function handleSearch(newQuery) {
    // Urgent: immediately update the input display
    setInputValue(newQuery);

    // Non-urgent: defer the expensive results update
    startTransition(() => {
      setResults(computeResults(newQuery)); // can be interrupted
    });
  }

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {/* If a new input comes while computing results: React interrupts
          and starts fresh with the new query */}
      <ResultsList results={results} />
    </>
  );
}
```

### `useDeferredValue` — Showing Stale Content During Transitions

```jsx
import { useDeferredValue } from 'react';

function SearchPage({ query }) {
  // Deferred: results may lag behind query
  const deferredQuery = useDeferredValue(query);

  const isStale = query !== deferredQuery;

  return (
    <div>
      <input value={query} ... />
      {/* Show stale results with visual indication while computing fresh ones */}
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        <Results query={deferredQuery} />
      </div>
    </div>
  );
}

// isStale=true: Results is still rendering old query → show faded
// isStale=false: Results finished → show at full opacity
```

---

## 7. Building a Minimal VDOM

Understanding VDOM by implementing a minimal version:

```javascript
// STEP 1: Create virtual DOM elements
function h(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children
        .flat()
        .map((child) =>
          typeof child === "string" || typeof child === "number"
            ? createTextNode(child)
            : child,
        ),
    },
  };
}

function createTextNode(text) {
  return {
    type: "TEXT_NODE",
    props: { textContent: String(text), children: [] },
  };
}

// STEP 2: Render virtual DOM to real DOM
function render(vnode, container) {
  const dom = createDom(vnode);
  container.appendChild(dom);
}

function createDom(vnode) {
  if (vnode.type === "TEXT_NODE") {
    return document.createTextNode(vnode.props.textContent);
  }

  const dom = document.createElement(vnode.type);

  // Apply props
  Object.entries(vnode.props || {}).forEach(([key, value]) => {
    if (key === "children") return;
    if (key.startsWith("on")) {
      dom.addEventListener(key.slice(2).toLowerCase(), value);
    } else {
      dom[key] = value;
    }
  });

  // Render children
  vnode.props.children.forEach((child) => {
    dom.appendChild(createDom(child));
  });

  return dom;
}

// STEP 3: Reconcile (diff old vs new virtual trees)
function reconcile(container, oldVNode, newVNode) {
  if (!oldVNode) {
    // Mount: new node
    container.appendChild(createDom(newVNode));
    return newVNode;
  }

  if (!newVNode) {
    // Unmount: old node removed
    container.removeChild(oldVNode._dom);
    return null;
  }

  if (oldVNode.type !== newVNode.type) {
    // Different type: replace entirely
    const newDom = createDom(newVNode);
    container.replaceChild(newDom, oldVNode._dom);
    newVNode._dom = newDom;
    return newVNode;
  }

  // Same type: update props and recurse into children
  const dom = oldVNode._dom;
  updateProps(dom, oldVNode.props, newVNode.props);
  newVNode._dom = dom;

  // Reconcile children (simplified — no key handling)
  const oldChildren = oldVNode.props.children || [];
  const newChildren = newVNode.props.children || [];
  const maxLen = Math.max(oldChildren.length, newChildren.length);

  for (let i = 0; i < maxLen; i++) {
    reconcile(dom, oldChildren[i], newChildren[i]);
  }

  return newVNode;
}

function updateProps(dom, oldProps, newProps) {
  // Remove old props
  Object.keys(oldProps || {}).forEach((key) => {
    if (key === "children") return;
    if (!(key in (newProps || {}))) {
      if (key.startsWith("on")) {
        dom.removeEventListener(key.slice(2).toLowerCase(), oldProps[key]);
      } else {
        dom[key] = "";
      }
    }
  });

  // Add/update new props
  Object.entries(newProps || {}).forEach(([key, value]) => {
    if (key === "children") return;
    if (key.startsWith("on")) {
      dom.addEventListener(key.slice(2).toLowerCase(), value);
    } else {
      dom[key] = value;
    }
  });
}
```

---

## 8. React Server Components — Beyond VDOM

React Server Components (RSC) challenge the VDOM model by rendering components on the server without sending their JavaScript to the client.

```
TRADITIONAL REACT (VDOM on client):
  Server → HTML → Client downloads ALL component code → hydrates
  Client manages virtual DOM for all components

REACT SERVER COMPONENTS:
  Server renders Server Components → RSC payload (not HTML, not JS)
  Client receives: static HTML + serialized React tree + Client Component JS
  Server Components: NEVER sent to client (zero JS bundle cost)
  Client Components: still use virtual DOM on client

RSC PAYLOAD FORMAT (simplified):
  "J0": ["$", "div", null, {"children": [
    ["$", "h1", null, {"children": "Server Data"}],
    "$L1"  ← reference to client component
  ]}]
  "J1": ClientComponent's serialized props
```

```jsx
// Server Component: runs on server, never shipped to client
// app/products/page.tsx (Next.js 13+ App Router)
async function ProductsPage() {
  // This fetch runs on the server — no useEffect, no loading state
  const products = await db.products.findMany();

  return (
    <main>
      <h1>Products</h1>
      {products.map((product) => (
        // Server Component renders HTML directly
        <ProductCard key={product.id} product={product} />
      ))}
      {/* Client Component: still uses VDOM, JavaScript shipped to client */}
      <AddToCartButton />
    </main>
  );
}

// Client Component: standard React VDOM behavior
("use client");
function AddToCartButton() {
  const [clicked, setClicked] = useState(false);
  return <button onClick={() => setClicked(true)}>Add to Cart</button>;
}
```

---

## 9. Alternatives to Virtual DOM

### Svelte — Compiled Reactivity (No VDOM)

```javascript
// Svelte: compiler generates direct DOM updates at build time
// No virtual DOM at runtime — the compiled output IS the update logic

// Svelte source:
let count = 0;
<button on:click={() => count++}>Count: {count}</button>;

// Compiled output (simplified):
function create_fragment(ctx) {
  let button;
  let t;

  return {
    c() {
      // create
      button = element("button");
      t = text(`Count: ${ctx[0]}`);
      append(button, t);
    },
    m(target) {
      // mount
      insert(target, button, null);
      listen(button, "click", ctx[1]); // ctx[1] = () => count++
    },
    p(ctx, dirty) {
      // update (patch)
      if (dirty & 1) {
        // bit 1 = count changed
        set_data(t, `Count: ${ctx[0]}`); // direct DOM update
      }
    },
  };
}
// No VDOM allocation, no diffing — direct DOM manipulation
```

### SolidJS — Fine-Grained Reactive (No VDOM)

```javascript
// SolidJS: signals track exactly which DOM nodes depend on which state
// Updates are surgical: only the specific DOM nodes that changed

import { createSignal, createEffect } from "solid-js";

const [count, setCount] = createSignal(0);

// At compile time, SolidJS detects: this text node depends on count
// At runtime: when count changes, ONLY this text node updates
// No diffing, no virtual tree

function Counter() {
  return (
    <button onClick={() => setCount((c) => c + 1)}>
      Count: {count()} {/* → direct fine-grained DOM update */}
    </button>
  );
}
// When count changes: no component re-render, no virtual DOM diff
// Just: textNode.textContent = `Count: ${count()}` — one DOM operation
```

### Performance Comparison

```
VIRTUAL DOM (React):
  On state change: re-render component → create new vdom → diff → patch DOM
  Cost: O(component subtree size) for diffing
  Predictable: always pays diffing cost, gains batching
  Good for: complex UIs with frequent bulk state changes

COMPILED REACTIVITY (Svelte):
  On state change: call specific compiled update function → patch DOM
  Cost: O(number of changed DOM nodes)
  Excellent for: UIs where change is predictable and granular

FINE-GRAINED REACTIVE (SolidJS):
  On state change: notify signal → update only dependent DOM nodes
  Cost: O(number of signals that changed)
  Excellent for: high-frequency updates, real-time data
  Overhead: signal tracking at each read
```

---

## 10. Good Practices

### ✅ Use stable keys from data — never array index

```jsx
// ✅ Stable keys → efficient reconciliation
items.map((item) => <Item key={item.id} {...item} />);
```

### ✅ Keep component identity stable

```jsx
// ✅ Component function defined outside render — stable identity
const ItemRenderer = memo(({ item }) => <div>{item.name}</div>);

function List({ items }) {
  return items.map((item) => <ItemRenderer key={item.id} item={item} />);
}

// ❌ Component defined inside render — new identity every render
function List({ items }) {
  const ItemRenderer = ({ item }) => <div>{item.name}</div>; // NEW function every render
  return items.map((item) => <ItemRenderer key={item.id} item={item} />);
  // → React sees new component type → unmounts and remounts every item!
}
```

### ✅ Use `key` to force remount when needed

```jsx
// ✅ Force fresh state by changing key
<Editor key={documentId} /> // New documentId → Editor remounts → fresh state
```

### ✅ Prefer CSS transitions over component state transitions for simple animations

```jsx
// ✅ CSS handles the transition, no VDOM work during animation
<div className={`card ${isOpen ? "expanded" : ""}`}>
  {/* CSS: .card.expanded { height: 200px; } with transition */}
</div>
// Component only re-renders once when isOpen changes
// The actual animation runs on the compositor thread — no JS per frame
```

---

## 11. Bad Practices

### ❌ Inline component definitions in JSX

```jsx
// ❌ Creates new component type every render → remount
function Parent({ data }) {
  const Child = () => <span>{data.value}</span>; // new function each render!
  return <Child />;
}

// ✅ Define outside:
const Child = ({ value }) => <span>{value}</span>;
function Parent({ data }) {
  return <Child value={data.value} />;
}
```

### ❌ Producing new object references unnecessarily

```jsx
// ❌ New object every render → breaks React.memo
<Component
  config={{ theme: "dark", size: "large" }} // new object every render!
  onClick={() => doSomething()} // new function every render!
/>;

// ✅ Stable references
const CONFIG = { theme: "dark", size: "large" }; // stable constant
function Parent() {
  const handleClick = useCallback(() => doSomething(), []);
  return <Component config={CONFIG} onClick={handleClick} />;
}
```

---

## 12. Common Mistakes

### Mistake 1 — Misunderstanding when re-renders happen

```jsx
// ❌ Thinking React only re-renders when the specific state changed
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
      <Child />{" "}
      {/* Re-renders on EVERY parent render — even if Child has no props! */}
    </div>
  );
}

// React re-renders ALL children when a parent re-renders (unless memoized)
// Use React.memo to prevent unnecessary child re-renders
const Child = React.memo(() => <div>Child</div>);
```

### Mistake 2 — Confusing reconciliation with DOM mutation

```jsx
// React re-rendering (reconciliation) ≠ DOM mutation

// Re-render means: the component function runs again
// DOM mutation means: a real DOM node was changed

// Re-render WITHOUT DOM mutation:
// - Props are same, computed virtual DOM is same
// - React reconciles but finds nothing to update
// - No DOM mutations happen (efficient)

// Re-render WITH DOM mutation:
// - Props changed, computed virtual DOM is different
// - React finds the diff and applies minimal DOM mutations
```

### Mistake 3 — Key must be unique among siblings, not globally

```jsx
// ✅ Keys only need to be unique within a list, not globally
function App() {
  return (
    <>
      {/* Key 1 here... */}
      {itemsA.map((item) => (
        <div key={item.id}>{item.name}</div>
      ))}

      {/* ...and key 1 here is fine — different parent */}
      {itemsB.map((item) => (
        <div key={item.id}>{item.name}</div>
      ))}
    </>
  );
}
// If itemsA has id:1 and itemsB has id:1: no conflict
// Keys are scoped to their sibling group
```

---

## 13. Interview-Level Explanation

> **"How does the virtual DOM work? Why does React use it? What are its limitations?"**

**Strong answer:**

> "The virtual DOM is a lightweight JavaScript object tree that mirrors the real DOM. When state changes, React calls the component function to produce a new virtual DOM tree, diffs it against the previous virtual tree using its reconciliation algorithm, identifies the minimum set of real DOM mutations needed, and applies them in a single commit phase.
>
> React's reconciliation uses O(n) heuristics rather than the optimal O(n³) diff algorithm. Two key heuristics: if element types differ between old and new trees, React unmounts the old subtree entirely and mounts the new one. Second, list items need keys so React can match items across renders — without keys, React compares by position, which means inserting an item at the start causes every subsequent item to appear "changed."
>
> The benefit of virtual DOM is that it acts as an automatic batching layer — component state changes accumulate as virtual DOM updates, which are then applied to the real DOM in one coordinated pass. React 18's fiber architecture made this interruptible: long reconciliations can be paused to let high-priority work (user input, animations) proceed.
>
> The limitations: virtual DOM isn't magic performance. Every re-render still calls the component function and creates JavaScript objects. For large component trees, this can be significant — which is why React.memo, useMemo, and useCallback exist. The virtual DOM also doesn't help with cases where direct DOM manipulation is genuinely the right approach, like canvas animations.
>
> Alternatives like SolidJS take a different approach: fine-grained reactivity where signals track exactly which DOM nodes depend on which pieces of state, and only those nodes update when state changes — no diffing at all. Svelte takes another approach: the compiler generates specific, direct DOM update functions at build time. Both eliminate the virtual DOM runtime overhead. The tradeoff is that React's virtual DOM is a more general model that handles complex state transitions and is easier to reason about at large scale."

---

## 14. Exercises

### Exercise 1 — Predict the reconciliation

For each pair of old/new JSX, describe what DOM operations React performs:

```jsx
// Scenario 1:
// Before:
<div>
  <p className="red">Hello</p>
  <span>World</span>
</div>

// After:
<div>
  <p className="blue">Hello</p>
  <span>World</span>
</div>
```

```jsx
// Scenario 2:
// Before:
<div>
  <Counter />
</div>

// After:
<span>
  <Counter />
</span>
```

```jsx
// Scenario 3 (with keys):
// Before: items = [A, B, C]
<ul>
  <li key="a">A</li>
  <li key="b">B</li>
  <li key="c">C</li>
</ul>

// After: items = [B, A, C] (A and B swapped)
<ul>
  <li key="b">B</li>
  <li key="a">A</li>
  <li key="c">C</li>
</ul>
```

<details>
<summary>Answers</summary>

```
Scenario 1:
  DOM operations: 1 (update className on p from "red" to "blue")
  React: div same → p same type → className changed → updateAttribute(p, className, "blue")
  span same → no change
  Total: 1 attribute update

Scenario 2:
  DOM operations: Unmount div + Counter, mount span + Counter
  React: div ≠ span → different type → destroy entire subtree
    → remove div from DOM
    → Counter unmounts (componentWillUnmount / useEffect cleanup)
    → Counter loses ALL state
  Then: mount new span → mount Counter (fresh state)
  This is a destructive operation — if Counter had local state: lost

Scenario 3 (with keys):
  DOM operations: 2 (reorder only)
  React: matches by key
    key "b": B existed → still exists → compare props (same) → no update, but position changed
    key "a": A existed → still exists → compare props (same) → no update, but position changed
    key "c": C existed → still exists → same position → nothing

  React moves DOM nodes: move <li key="b"> before <li key="a">
  Result: 1 DOM insertBefore operation (reorder b before a)
  Counter state is preserved (nodes moved, not remounted)
```

</details>

---

## 🔗 Related Topics

- [`rendering/01-dom-batching.md`](./01-dom-batching.md) — Virtual DOM as a batching layer
- [`browser-internals/02-dom-tree-creation.md`](../browser-internals/02-dom-tree-creation.md) — Real DOM that virtual DOM represents
- [`performance/07-memoization.md`](../performance/07-memoization.md) — React.memo, useMemo for VDOM optimization
- [`rendering/03-cooperative-scheduling.md`](./03-cooperative-scheduling.md) — Fiber's scheduler in depth

---

<div align="center">

**Next:** [`rendering/03-cooperative-scheduling.md`](./03-cooperative-scheduling.md) →

</div>
