# 03 — Challenge: Build a Mini React from Scratch

> **"There is no faster way to understand why React behaves the way it does than building a tiny version of it yourself. Every 'gotcha' you've ever hit — why hooks need a stable call order, why keys matter in lists, why setState is asynchronous — stops being a memorized rule and becomes an obvious consequence of how the thing is built, once you've built one."**

This challenge builds a minimal but real implementation of React's core mechanics: a `createElement` function, a renderer that turns elements into DOM nodes, a reconciliation algorithm that diffs and updates efficiently, and a hooks system with `useState` and `useEffect`. By the end, you'll understand React's fundamental architecture from first principles.

---

## The Goal

Build `MiniReact` — a library with `createElement`, `render`, function components, `useState`, and `useEffect` — capable of rendering and updating a real interactive UI, including list reconciliation with keys.

---

## Stage 1 — createElement and Virtual DOM

### Requirements

- `createElement(type, props, ...children)` returns a plain JavaScript object representing a virtual DOM node
- Support both DOM element types (`'div'`, `'span'`) and function components
- Support text nodes (strings/numbers as children)

<details>
<summary>Hints</summary>

- A virtual DOM element is just `{ type, props: { ...props, children } }`
- Text content needs to be wrapped in a special element type so the renderer can treat it uniformly with other elements
- `children` can be a mix of elements, strings, numbers, arrays (from `.map()`), and `null`/`false` (conditional rendering) — flatten and filter these
</details>

<details>
<summary>Solution</summary>

```javascript
const TEXT_ELEMENT = "TEXT_ELEMENT";

function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children
        .flat(Infinity) // flatten nested arrays (from .map())
        .filter((child) => child != null && child !== false && child !== true) // remove null/false/true
        .map((child) =>
          typeof child === "object" ? child : createTextElement(child),
        ),
    },
  };
}

function createTextElement(text) {
  return {
    type: TEXT_ELEMENT,
    props: { nodeValue: String(text), children: [] },
  };
}

// Usage (this is what JSX compiles to):
const element = createElement(
  "div",
  { className: "greeting" },
  createElement("h1", null, "Hello"),
  "World",
);

// Equivalent JSX (if you set up a JSX pragma pointing to MiniReact.createElement):
// /** @jsx MiniReact.createElement */
// const element = <div className="greeting"><h1>Hello</h1>World</div>;
```

**Why this matters:** This is the data structure everything else builds on. Notice it's just plain objects — no DOM, no class instances, nothing expensive. This is exactly why creating virtual DOM elements is cheap: `createElement` does essentially no work beyond building a tree of plain objects. The actual expense in React comes later, in reconciliation and DOM mutation — never in element creation itself.

</details>

---

## Stage 2 — The Renderer (Virtual DOM → Real DOM)

### Requirements

- `render(element, container)` creates real DOM nodes from a virtual DOM tree and appends them to `container`
- Support setting props (attributes) and event listeners (`onClick`, etc.)
- Recursively render children

<details>
<summary>Hints</summary>

- `TEXT_ELEMENT` types become `document.createTextNode()`, everything else becomes `document.createElement()`
- Prop names starting with "on" are event listeners (strip the "on", lowercase it, use `addEventListener`)
- The `children` prop is special — don't set it as a DOM attribute, recurse into it instead
</details>

<details>
<summary>Solution</summary>

```javascript
function createDom(element) {
  const dom =
    element.type === TEXT_ELEMENT
      ? document.createTextNode("")
      : document.createElement(element.type);

  updateDom(dom, {}, element.props); // apply initial props (oldProps = {})

  return dom;
}

const isEvent = (key) => key.startsWith("on");
const isProperty = (key) => key !== "children" && !isEvent(key);
const isNew = (prev, next) => (key) => prev[key] !== next[key];
const isGone = (prev, next) => (key) => !(key in next);

function updateDom(dom, prevProps, nextProps) {
  // Remove old event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter((key) => !(key in nextProps) || isNew(prevProps, nextProps)(key))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.removeEventListener(eventType, prevProps[name]);
    });

  // Remove old properties
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = "";
    });

  // Set new or changed properties
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = nextProps[name];
    });

  // Add new event listeners
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.addEventListener(eventType, nextProps[name]);
    });
}

function render(element, container) {
  const dom = createDom(element);

  element.props.children.forEach((child) => render(child, dom));

  container.appendChild(dom);
}

// Usage:
const element = createElement(
  "div",
  { className: "app" },
  createElement("h1", null, "Hello MiniReact"),
);
render(element, document.getElementById("root"));
```

**What breaks next:** Calling `render()` again to update the UI doesn't update anything — it just appends MORE DOM nodes on top of the old ones. There's no concept of "diffing against the previous tree" yet. That's the entire point of Stage 3.

</details>

---

## Stage 3 — Reconciliation (The Diffing Algorithm)

### The Problem This Introduces

Real applications update the same UI repeatedly. Re-creating the entire DOM tree on every update is correct but catastrophically slow — and it would lose input focus, scroll position, and CSS transition state on every single update. You need to compare the new virtual tree to the previous one and apply only the minimal necessary DOM changes.

### Requirement

Implement a `fiber`-inspired reconciliation pass that:

- Compares old and new virtual trees node by node
- UPDATEs DOM nodes whose type is the same (patches props)
- REPLACEs DOM nodes whose type changed (unmount old, mount new)
- DELETEs DOM nodes that no longer exist in the new tree

<details>
<summary>Hints</summary>

- You need to keep a reference to "what was rendered last time" to compare against
- A simplified, NON-fiber, purely recursive approach is acceptable for this challenge (skip the interruptible-rendering complexity of real React) — the goal is to understand the DIFFING ALGORITHM, not the scheduler
- Three cases per node: same type (update), different type (replace), node doesn't exist in new tree (delete) or didn't exist in old tree (create)
</details>

<details>
<summary>Solution</summary>

```javascript
let currentRoot = null; // the previously rendered virtual tree, with DOM refs attached

function reconcile(parentDom, oldNode, newNode) {
  if (!oldNode && newNode) {
    // CREATE: didn't exist before, exists now
    const dom = createDom(newNode);
    newNode.dom = dom;
    newNode.props.children.forEach((child) => reconcile(dom, null, child));
    parentDom.appendChild(dom);
    return newNode;
  }

  if (oldNode && !newNode) {
    // DELETE: existed before, doesn't exist now
    parentDom.removeChild(oldNode.dom);
    return null;
  }

  if (oldNode.type !== newNode.type) {
    // REPLACE: type changed — can't patch, must unmount + remount
    const dom = createDom(newNode);
    newNode.dom = dom;
    newNode.props.children.forEach((child) => reconcile(dom, null, child));
    parentDom.replaceChild(dom, oldNode.dom);
    return newNode;
  }

  // UPDATE: same type — patch props, recurse into children
  newNode.dom = oldNode.dom; // reuse the existing DOM node!
  updateDom(newNode.dom, oldNode.props, newNode.props);

  reconcileChildren(
    newNode.dom,
    oldNode.props.children,
    newNode.props.children,
  );

  return newNode;
}

function reconcileChildren(parentDom, oldChildren, newChildren) {
  const maxLength = Math.max(oldChildren.length, newChildren.length);
  for (let i = 0; i < maxLength; i++) {
    reconcile(parentDom, oldChildren[i], newChildren[i]);
  }
}

function render(element, container) {
  const newRoot = reconcile(container, currentRoot, element);
  currentRoot = newRoot;
}

// Now calling render() AGAIN with an updated element tree correctly
// patches the existing DOM instead of duplicating it:
let count = 0;
function renderApp() {
  const element = createElement(
    "div",
    null,
    createElement("p", null, `Count: ${count}`),
    createElement(
      "button",
      {
        onClick: () => {
          count++;
          renderApp();
        },
      },
      "Increment",
    ),
  );
  render(element, document.getElementById("root"));
}
renderApp();
```

**The critical bug this naive version has:** `reconcileChildren` compares children by INDEX (`oldChildren[i]` vs `newChildren[i]`). If you reorder a list — say, sorting an array of items — every item appears to have "changed" because the item that WAS at index 0 is now at index 2. This causes unnecessary DOM updates (and worse, loses input focus / animation state on reordered items because React would think the DOM node represents a DIFFERENT logical item now). This is EXACTLY why React requires `key` props on list items — Stage 4 fixes this.

</details>

---

## Stage 4 — Keys and List Reconciliation

### Requirement

Use `key` props to match children across renders by IDENTITY rather than by index, so reordering a list reuses the correct DOM nodes instead of patching by position.

<details>
<summary>Hints</summary>

- Build a Map from `key` to old child, for nodes that have a key
- For each new child: if it has a matching key in the old children, reconcile against THAT old child (regardless of position) — not the old child at the same index
- Children without keys still fall back to index-based matching (with a console warning, just like real React)
</details>

<details>
<summary>Solution</summary>

```javascript
function reconcileChildren(parentDom, oldChildren, newChildren) {
  // Build a lookup of old children BY KEY (for children that have one)
  const oldChildrenByKey = new Map();
  const oldChildrenWithoutKey = [];

  oldChildren.forEach((child, index) => {
    const key = child?.props?.key;
    if (key != null) {
      oldChildrenByKey.set(key, child);
    } else {
      oldChildrenWithoutKey.push({ child, index });
    }
  });

  let unkeyedIndex = 0;

  newChildren.forEach((newChild, newIndex) => {
    const key = newChild?.props?.key;
    let matchedOldChild = null;

    if (key != null && oldChildrenByKey.has(key)) {
      // MATCH BY KEY: find the SAME logical item, regardless of position
      matchedOldChild = oldChildrenByKey.get(key);
      oldChildrenByKey.delete(key); // consumed — won't be deleted later
    } else if (key == null) {
      // Fall back to index-based matching for unkeyed children
      matchedOldChild = oldChildrenWithoutKey[unkeyedIndex]?.child ?? null;
      unkeyedIndex++;
    }

    reconcile(parentDom, matchedOldChild, newChild);

    // If the matched old child existed at a DIFFERENT position, the DOM
    // node needs to be MOVED, not just patched, to preserve order:
    if (matchedOldChild && matchedOldChild.dom) {
      const currentDomAtIndex = parentDom.childNodes[newIndex];
      if (currentDomAtIndex !== matchedOldChild.dom) {
        parentDom.insertBefore(matchedOldChild.dom, currentDomAtIndex || null);
      }
    }
  });

  // Any old keyed children NOT consumed above no longer exist — delete them
  oldChildrenByKey.forEach((staleChild) => {
    parentDom.removeChild(staleChild.dom);
  });
}

// Demonstration: reordering now correctly REUSES DOM nodes by key,
// instead of patching every position's content
function renderList(items) {
  const element = createElement(
    "ul",
    null,
    ...items.map((item) => createElement("li", { key: item.id }, item.label)),
  );
  render(element, document.getElementById("root"));
}

renderList([
  { id: "a", label: "A" },
  { id: "b", label: "B" },
  { id: "c", label: "C" },
]);
// DOM: <li>A</li><li>B</li><li>C</li> (3 li elements created)

renderList([
  { id: "c", label: "C" },
  { id: "a", label: "A" },
  { id: "b", label: "B" },
]);
// WITHOUT keys: all 3 li elements get their TEXT CONTENT patched (A→C, B→A, C→B)
//   — if any li had focus or an active CSS transition, it's now associated
//   with the WRONG logical item
// WITH keys (this implementation): the EXISTING DOM nodes for a, b, c are
//   REORDERED via insertBefore — same 3 DOM nodes, just moved, so any
//   focus/transition state stays correctly attached to its logical item
```

**Why this exactly mirrors React's behavior:** This is conceptually identical to what React's reconciler does — match by key when present, fall back to index when absent, and physically move (not recreate) DOM nodes that matched by key but changed position. This is also why React's warning "each child in a list should have a unique key prop" exists: without keys, the index-fallback path can produce subtly wrong behavior exactly like the unkeyed example above.

</details>

---

## Stage 5 — Function Components and Hooks (useState)

### Requirement

Support function components (functions that return elements) and implement `useState` with correct re-render-on-update behavior.

<details>
<summary>Hints</summary>

- A function component node has `type` = the function itself, not a string
- When reconciling: if `type` is a function, CALL it with props to get the actual element tree, then reconcile that
- `useState` needs to know WHICH component instance and WHICH hook call index it's being invoked from — this requires tracking a "currently rendering" pointer and a per-component hooks array
- This is the trickiest part: hooks rely on CALL ORDER, which is why conditional hooks break things
</details>

<details>
<summary>Solution</summary>

```javascript
let wipFiber = null; // the fiber (component instance) currently being rendered
let hookIndex = null; // index into wipFiber.hooks for the NEXT hook call

function useState(initial) {
  const oldHook = wipFiber.alternate?.hooks?.[hookIndex];
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [], // pending state updates (actions)
  };

  // Apply any queued actions from a previous setState call
  const actions = oldHook ? oldHook.queue : [];
  actions.forEach((action) => {
    hook.state = typeof action === "function" ? action(hook.state) : action;
  });

  const setState = (action) => {
    hook.queue.push(action);
    // Schedule a re-render of the WHOLE tree from the root
    // (real React schedules more granularly — this is the simplified version)
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    };
    nextUnitOfWork = wipRoot;
    performWorkLoop();
  };

  wipFiber.hooks.push(hook);
  hookIndex++;
  return [hook.state, setState];
}

function updateFunctionComponent(fiber) {
  wipFiber = fiber;
  hookIndex = 0;
  wipFiber.hooks = []; // fresh hooks array for THIS render

  // Call the function component, passing props, to get the element tree
  const children = [fiber.type(fiber.props)];
  reconcileChildren(fiber, children);
}

// Simplified single-pass work loop (real React uses a fiber-based,
// interruptible loop — this version is synchronous/recursive for clarity)
function performWorkLoop() {
  function walk(fiber) {
    if (typeof fiber.type === "function") {
      updateFunctionComponent(fiber);
    } else {
      updateHostComponent(fiber);
    }
    fiber.children?.forEach(walk);
  }
  walk(wipRoot);
  commitRoot();
}

// Usage:
function Counter() {
  const [count, setCount] = useState(0);
  return createElement(
    "div",
    null,
    createElement("p", null, `Count: ${count}`),
    createElement(
      "button",
      { onClick: () => setCount((c) => c + 1) },
      "Increment",
    ),
  );
}

render(createElement(Counter), document.getElementById("root"));
```

**Why hook call order matters — now it's obvious, not a rule to memorize:** `useState` reads from `wipFiber.hooks[hookIndex]` and then increments `hookIndex`. There's no name, no key, no identifier tying a specific `useState` call to its stored value — ONLY the call order within a single render. If a component calls `useState` conditionally (sometimes 2 calls, sometimes 3), then on a render where only 2 hooks are called, hook index 2's stored state (meant for a DIFFERENT logical piece of state) gets silently assigned to whatever the 2nd `useState` call asks for. This is precisely why React's Rules of Hooks forbid conditional hook calls — you've now built the exact mechanism that makes that rule necessary, not just memorized that it's a rule.

</details>

---

## Stage 6 — useEffect

### Requirement

Implement `useEffect(callback, deps)` that runs the callback after the DOM has been updated, and only re-runs when dependencies change (shallow comparison), supporting cleanup functions.

<details>
<summary>Hints</summary>

- Effects must run AFTER the DOM commit, not during the render/reconciliation pass
- Store effects per-fiber (like hooks), compare new deps against old deps from `fiber.alternate`
- Cleanup: call the PREVIOUS effect's returned cleanup function before running the NEW effect (and on unmount)
</details>

<details>
<summary>Solution</summary>

```javascript
function useEffect(callback, deps) {
  const oldHook = wipFiber.alternate?.hooks?.[hookIndex];

  const hasChanged =
    !oldHook || !deps || deps.some((dep, i) => dep !== oldHook.deps?.[i]);

  const hook = {
    callback,
    deps,
    cleanup: oldHook?.cleanup, // carry forward in case we don't run this time
    hasChanged,
  };

  wipFiber.hooks.push(hook);
  hookIndex++;

  // Queue this for execution AFTER commit (don't run during render!)
  if (hasChanged) {
    pendingEffects.push(hook);
  }
}

let pendingEffects = [];

function commitRoot() {
  // ... commit DOM mutations first ...
  commitWork(wipRoot.child);

  currentRoot = wipRoot;
  wipRoot = null;

  // THEN run effects, after the DOM is fully updated (mirrors React's
  // "commit phase, then run passive effects" ordering)
  flushEffects();
}

function flushEffects() {
  const effectsToRun = pendingEffects;
  pendingEffects = [];

  effectsToRun.forEach((hook) => {
    if (hook.cleanup) hook.cleanup(); // clean up the PREVIOUS effect first
    hook.cleanup = hook.callback(); // run the new effect, store its cleanup
  });
}

// Usage:
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => setSeconds((s) => s + 1), 1000);
    return () => clearInterval(interval); // cleanup
  }, []); // empty deps — runs once

  return createElement("p", null, `Seconds: ${seconds}`);
}
```

**Why effects must run after commit, not during render:** If `useEffect` callbacks ran DURING the render/reconciliation pass, they could read DOM properties that haven't been updated yet, or cause additional state updates that interfere with the render in progress. Separating "compute what the DOM should look like" (render phase) from "make the DOM look like that" (commit phase) from "run side effects now that the DOM is updated" (effect phase) is fundamental to React's whole architecture — and it's the reason `useLayoutEffect` (runs synchronously after commit, before paint) and `useEffect` (runs asynchronously after paint) are different hooks with different timing guarantees.

</details>

---

## Stage 7 — Stretch Goals

```
1. CONTEXT (useContext)
   Implement a Context system: createContext(), Provider component,
   and useContext() that reads the nearest ancestor Provider's value
   by walking up the fiber tree.

2. CONCURRENT/INTERRUPTIBLE RENDERING
   Replace the synchronous recursive walk in Stage 5 with a real
   fiber-based work loop using requestIdleCallback (or a scheduler
   library), so large renders can be interrupted by higher-priority
   updates (like user input) — this is what gives real React its name
   "Fiber" architecture.

3. ERROR BOUNDARIES
   Implement a way for a component to catch errors thrown by its
   children during render, and render a fallback UI instead.

4. useMemo and useCallback
   Implement these using the same hooks-array mechanism as useState —
   compare deps, return cached value/function if deps haven't changed.

5. BATCHED UPDATES
   Currently, every setState call triggers an immediate full re-render.
   Implement batching: multiple setState calls within the same synchronous
   execution (e.g., in one event handler) should only trigger ONE re-render,
   not one per call.
```

---

## What You Should Have Learned

```
1. The virtual DOM is just plain JavaScript objects — its value isn't
   "speed" inherently, it's providing a DECLARATIVE description of what
   the UI should look like, which then enables efficient diffing against
   the previous description.

2. Reconciliation by default matches children BY POSITION (index), which
   is why reordering lists without keys causes React to patch content at
   each position rather than moving DOM nodes — keys give React an
   identity to match against, independent of position.

3. Hooks work by relying ENTIRELY on call order within a render — there's
   no hook "name" stored anywhere, just a flat array indexed by call
   position. This is exactly why conditional hook calls break things:
   the array position no longer reliably corresponds to the same logical
   piece of state across renders.

4. The render phase (compute the new tree) and commit phase (mutate the
   real DOM) and effect phase (run side effects) are deliberately
   separated stages, each with different timing guarantees — this
   separation is what enables useLayoutEffect vs useEffect to have
   different semantics, and what enables (in real React) the render
   phase to be interruptible without leaving the DOM in an inconsistent
   state.

5. Building this from scratch turns "React's behavior" from a list of
   memorized rules into a set of logical consequences of a specific,
   understandable implementation — which is exactly the kind of
   understanding that lets you debug genuinely novel problems in
   production React applications.
```

---

## 🔗 Related Topics

- [`rendering/02-virtual-dom.md`](../rendering/02-virtual-dom.md)
- [`javascript-core/01-execution-context.md`](../javascript-core/01-execution-context.md)
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md)
- [`anti-patterns/04-stale-closures.md`](../anti-patterns/04-stale-closures.md)

---

<div align="center">

**`challenges/` section complete!** 🎉

All 3 challenges files done:
`01-build-a-virtualized-list.md` · `02-build-a-state-management-library.md` · **`03-build-a-mini-react.md`** ✓

**Next section:** [`projects/`](../projects/) →

</div>
