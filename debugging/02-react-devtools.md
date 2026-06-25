# 02 — React DevTools

> **"React DevTools is what turns 'my component is re-rendering too much' from a feeling into a measurement. Without it you're guessing at the component tree, guessing at state, guessing at what triggered a render. With it, every question has a precise answer."**

React DevTools extends Chrome DevTools with React-specific panels for inspecting the component tree, examining props and state, profiling renders, and understanding why a component updated. It makes the virtual DOM visible. This document covers the Components panel for live inspection, the Profiler for measuring render performance, debugging common React issues, and the hooks and state inspection features that replace print debugging.

---

## 📚 Table of Contents

1. [Installation and Setup](#1-installation-and-setup)
2. [Components Panel — Tree and State Inspection](#2-components-panel--tree-and-state-inspection)
3. [Inspecting Props, State, and Hooks](#3-inspecting-props-state-and-hooks)
4. [Context Inspection](#4-context-inspection)
5. [Profiler Panel — Measuring Renders](#5-profiler-panel--measuring-renders)
6. [Why Did This Render?](#6-why-did-this-render)
7. [Debugging Common React Issues](#7-debugging-common-react-issues)
8. [Programmatic Debugging with DevTools Hooks](#8-programmatic-debugging-with-devtools-hooks)
9. [Good Practices](#9-good-practices)
10. [Common Debugging Workflows](#10-common-debugging-workflows)
11. [Interview-Level Explanation](#11-interview-level-explanation)

---

## 1. Installation and Setup

```
INSTALLATION:
  Chrome: install "React Developer Tools" from Chrome Web Store
  Firefox: install "React Developer Tools" from Firefox Add-ons
  Edge: same Chrome extension works

LOCATING THE PANELS:
  After installing: two new tabs in Chrome DevTools
  "⚛ Components": component tree + state inspection
  "⚛ Profiler": render timing and flame chart

VERIFYING IT'S WORKING:
  On a React 16.8+ site: colored React logo in browser toolbar (blue = dev build)
  Grey logo: production build (some features limited)
  No logo: not a React app

SETTINGS (gear icon in DevTools):
  "Highlight updates when components render" (extremely useful)
    → Blue flash on components when they render
    → Shows which parts of the UI re-render on every interaction
  "Reload and profile" → records from page load (for startup performance)
  Display density: compact or comfortable
  Theme: dark/light
```

---

## 2. Components Panel — Tree and State Inspection

```
COMPONENT TREE (left panel):
  Shows the React virtual DOM tree
  Components displayed by their function/class name
  Context providers and consumers labeled (e.g., ThemeContext.Provider)
  Anonymous components: shown as "Anonymous" — name them for clarity

NAVIGATION:
  Click component → shows details in right panel
  Arrow keys: navigate up/down in tree
  Search (Ctrl+F): filter components by name
  Select element: pick icon (arrow) → click element in page → jumps to component

TREE COLLAPSING:
  Long lists of same type: shown as "3 items..." (expand to see all)
  Hold Alt/Option while clicking triangle: expand/collapse entire subtree

COMPONENT SELECTION FROM PAGE:
  Click the target icon (cursor arrow) in DevTools toolbar
  Then click any element on the page
  → Jumps to the lowest React component that owns that element
  Much faster than finding components in a long tree
```

### Reading the Component Tree

```
WHAT THE TREE SHOWS:
  Only React components (not DOM nodes)
  Displayed in same nesting order as the virtual DOM
  The tree is live: updates in real time as state changes

TREE vs DOM:
  <div className="app">                ← DOM node (not in React tree)
    <Header />                         ← React component (shown in tree)
      <nav>                            ← DOM node
        <Logo />                       ← React component (shown in tree)
        <NavLinks />                   ← React component (shown in tree)
    <ProductPage>                      ← React component (shown in tree)
      <Suspense>                       ← shown as "Suspense" in tree
        <ProductGallery>               ← shown when loaded
```

---

## 3. Inspecting Props, State, and Hooks

```
RIGHT PANEL (when component is selected):

PROPS section:
  All props passed to this component
  Expand objects/arrays
  Values shown with types: strings in quotes, numbers plain, booleans checked

STATE section:
  All useState values for the component
  Named by hook call index (useState#0, useState#1) unless named via hook
  Expand complex state (objects, arrays)

HOOKS section:
  All hooks used by the component in order
  useState: shows current value
  useEffect: shows deps array
  useContext: shows current context value
  useRef: shows .current value
  useMemo: shows cached value
  Custom hooks: expanded into their constituent hooks

LIVE EDITING (only in development mode):
  Click any prop/state value → edit inline → Enter to apply
  Add properties: click "+" next to an object
  Great for: testing edge cases without modifying code
    "What happens if this user has no avatar URL?"
    "What if the list has 0 items?"
```

### Naming Hooks for DevTools

```javascript
// ❌ Anonymous: appears as "useState#0" in DevTools
const [isOpen, setIsOpen] = useState(false);
const [items, setItems] = useState([]);

// ✅ Named via useDebugValue in custom hooks
function useDisclosure(initial = false) {
  const [isOpen, setIsOpen] = useState(initial);

  // This label appears in React DevTools when inspecting the hook
  useDebugValue(isOpen ? "Open" : "Closed");

  return {
    isOpen,
    toggle: () => setIsOpen((o) => !o),
    close: () => setIsOpen(false),
  };
}

// ✅ Format function for expensive computations (only runs in DevTools, not prod)
function useUser(userId) {
  const user = useUserStore((s) => s.users[userId]);
  useDebugValue(user, (u) => `User: ${u?.name} (${userId})`);
  return user;
}
// DevTools shows "User: Alice (user-42)" instead of the raw user object
```

---

## 4. Context Inspection

```
CONTEXT IN THE TREE:
  Provider components appear as "ThemeContext.Provider" etc.
  Consumer hooks: useContext values visible in the component's Hooks section

FINDING ALL CONTEXT CONSUMERS:
  Search for the context name in the Components panel filter
  All components using that context appear in the tree

INSPECTING CONTEXT VALUE:
  Click the Provider → right panel shows the current value being provided
  Click any component → Hooks section shows its useContext values

DEBUGGING CONTEXT UPDATES:
  "Highlight updates when components render" shows which components
  re-render when context value changes
  If too many components flash: context value is changing unnecessarily
  → Check if value object is recreated on every parent render (needs useMemo)
```

---

## 5. Profiler Panel — Measuring Renders

```
RECORDING A PROFILE:
  Click blue circle (record) → interact with app → click again to stop
  OR: click "Reload and Profile" → reloads page, records from start

FLAME CHART (what you see):
  X axis: commits (each re-render cycle)
    Each bar = one commit
    Wider = more components rendered in that commit
    Color gradient (yellow → orange → red) = slower commits

  Y axis (inside each commit): component tree
    Each row = a component
    Width = render time
    Color:
      Grey = component did NOT render this commit (was bailed out)
      Color (yellow/orange) = component DID render this commit
      Darker = more time spent rendering

NAVIGATING:
  Click a commit bar at the top → show that commit's flame chart
  Hover a component bar → tooltip shows component name, render duration
  Double-click component → zoom into that subtree
  Click "⟳" to see what triggered this commit
```

### Ranked Chart

```
Profiler → switch from "Flamechart" to "Ranked"

RANKED CHART:
  Lists all components that rendered in a commit
  Sorted by render duration (slowest first)
  Shows: component name, render time, count of renders

USE CASE:
  Find the single slowest component in a commit at a glance
  "Which component is responsible for this 80ms commit?"
  → Ranked chart: top entry is the culprit
```

---

## 6. Why Did This Render?

```
ENABLING "WHY DID THIS RENDER?":
  Profiler Settings (gear) → "Record why each component rendered while profiling"
  Must be enabled BEFORE recording

READING THE INFORMATION:
  After recording: click any component that rendered in the profiler
  Bottom section shows "Why did this render?":
    "Props changed: { items: [Array(5)] → [Array(6)] }"
    "State changed: (count: 5 → 6)"
    "Context changed"
    "Hooks changed: (useState#0)"
    "Parent re-rendered"
    "First render"

THIS IS THE MOST VALUABLE FEATURE:
  "Why does ProductCard re-render when I interact with the search bar?"
  Record → interact → click ProductCard → "Why did this render?"
  → "Parent re-rendered" (ProductList re-renders → all children re-render)
  → FIX: React.memo on ProductCard (if ProductList must re-render)
  OR
  → "Props changed: { onClick: function → function }"
  → FIX: useCallback on the onClick handler in ProductList
```

### Interpreting "Parent re-rendered"

```
"Parent re-rendered" means:
  The parent re-rendered AND the child is not wrapped in React.memo
  The child re-rendered as a "free" consequence, even though its props didn't change

DECISION:
  Is the child expensive to render?
    YES: Add React.memo → then check if "Props changed" with unstable references
         If props are still unstable: add useMemo/useCallback in the parent
    NO: Leave it — re-rendering is cheap, optimization isn't worth the complexity

  Did the parent re-render for a reason unrelated to this child?
    YES: This is the core prop drilling / over-subscription problem
         Consider restructuring so the parent doesn't re-render unnecessarily
```

---

## 7. Debugging Common React Issues

### Issue 1 — Component Not Updating

```
SYMPTOM: State changed, but the component isn't showing the new value.

DIAGNOSE:
  1. Components panel → select the component
  2. Click through the Hooks/State section — did the state value change?

  CASE A: State DID change in DevTools but UI doesn't reflect it
    → CSS issue: the DOM is correct but CSS isn't showing it
    → Check Elements panel for the actual DOM node

  CASE B: State DID NOT change
    → The setState call isn't being reached
    → Add a breakpoint in the event handler → step through to find where it stops

  CASE C: State shows old value
    → Stale closure: the handler is calling setState on a stale captured value
    → See anti-patterns/04-stale-closures.md
```

### Issue 2 — Infinite Re-render Loop

```
SYMPTOM: "Maximum update depth exceeded" error in console.
         OR: page becomes unresponsive, fan starts spinning.

DIAGNOSE:
  1. Profiler → Record → wait 1 second → Stop
  2. Flame chart shows: many, many commits in rapid succession
  3. Click one commit → "Why did this render?"
  4. Usually shows: "State changed" on every render

  COMMON CAUSE: setState called inside render or in useEffect without deps

  EXAMPLES:
    useEffect(() => { setState(something); }); // no deps = runs on every render → sets state → re-renders → runs again
    useEffect(() => { setState(fn()); }, [fn]); // fn is a new function every render → deps always "change"

  FIX: add correct deps, use functional updater, or memoize the triggering value
```

### Issue 3 — Unexpected Re-renders After React.memo

```
SYMPTOM: Added React.memo but component still re-renders on every parent render.

DIAGNOSE:
  1. Profiler → Record → trigger parent re-render → Stop
  2. Click the memo'd component → "Why did this render?"
  3. If "Props changed": shows which specific prop changed
  4. Common answer: "Props changed: { config: [Object] → [Object] }"
     → `config` is a new object on every render

  FIX:
    Inline object prop → useMemo in parent
    Inline arrow function → useCallback in parent

  VERIFY: re-run Profiler → component should now show grey (skipped render)
```

---

## 8. Programmatic Debugging with DevTools Hooks

```javascript
// React DevTools exposes a global hook for programmatic access
// Use in console: window.__REACT_DEVTOOLS_GLOBAL_HOOK__

// Get the fiber root (requires internal knowledge — not stable API)
// Better approach: use the React DevTools bridge to expose component state

// Custom debugging in development mode
if (process.env.NODE_ENV === "development") {
  // Track render count for a specific component
  let renderCount = 0;
  const OriginalComponent = MyComponent;
  MyComponent = function DebuggedMyComponent(props) {
    renderCount++;
    console.log(`MyComponent rendered (${renderCount} times)`, props);
    return OriginalComponent(props);
  };
  MyComponent.displayName = "MyComponent";
}

// React.Profiler component: programmatic profiling (no DevTools needed)
import { Profiler } from "react";

function onRenderCallback(
  id, // the "id" prop of the Profiler tree that just committed
  phase, // 'mount' or 'update'
  actualDuration, // time rendering the committed update
  baseDuration, // estimated time to render entire subtree (no memoization)
  startTime, // when React started rendering this update
  commitTime, // when React committed this update
) {
  if (actualDuration > 16) {
    // 16ms = 1 frame at 60fps
    console.warn(`Slow render: ${id} took ${actualDuration.toFixed(2)}ms`);
  }
  analytics.track("render_performance", {
    id,
    phase,
    duration: actualDuration,
  });
}

<Profiler id="ProductList" onRender={onRenderCallback}>
  <ProductList products={products} />
</Profiler>;
```

---

## 9. Good Practices

### ✅ Always name your components

```javascript
// ❌ Anonymous functions in React devtools show as "Anonymous"
export default () => <div>App</div>;
const Button = React.memo(() => <button>Click</button>);

// ✅ Named functions are shown by name in the component tree
export default function App() { return <div>App</div>; }
const Button = React.memo(function Button() { return <button>Click</button>; });

// ✅ Set displayName for HOC-wrapped components
const EnhancedButton = withSomething(Button);
EnhancedButton.displayName = 'EnhancedButton';
```

### ✅ Use useDebugValue for custom hooks

```javascript
// ✅ Makes custom hooks self-documenting in DevTools
function usePermissions(userId) {
  const permissions = /* ... */;
  useDebugValue(permissions?.length ?? 0, (n) => `${n} permissions`);
  return permissions;
}
```

### ✅ Enable "Highlight updates" during development

```
DevTools → ⚛ Components → Settings (gear) → "Highlight updates when components render"
This makes unnecessary re-renders immediately visible without profiling
If the entire page flashes on every keystroke in a search box: something is wrong
```

---

## 10. Common Debugging Workflows

### Workflow 1 — "Why does X re-render?"

```
1. ⚛ Profiler → Settings → Enable "Record why each component rendered"
2. Record → trigger the interaction → Stop
3. Find the component in the flame chart (use search in Components panel)
4. Click it → "Why did this render?"
5. Act on the specific reason shown:
   "Parent re-rendered" → wrap in React.memo if expensive
   "Props changed: { fn: function → function }" → useCallback in parent
   "State changed" → check if the state change is necessary
   "Context changed" → split context if not all consumers need this update
```

### Workflow 2 — "This state seems wrong"

```
1. ⚛ Components → select the component
2. Check Hooks/State section: what is the actual current value?
3. If value is wrong: set a breakpoint in the setState call that should set it
4. Trigger the action → breakpoint fires
5. Check call stack → is this function being called?
6. Check the argument to setState → is the right value being passed?
7. If setState has the right value but DevTools shows wrong value:
   → Check if you're mutating state directly (should produce new object/array)
```

### Workflow 3 — "The component tree isn't what I expect"

```
1. ⚛ Components → use search to find your component
2. Check its parent chain → which components are above it?
3. Check props → are they what you expect?
4. Look for unexpected Providers in the tree → extra context wrappers?
5. Check if component is conditionally rendered → is it inside a conditional?
```

---

## 11. Interview-Level Explanation

> **"How do you use React DevTools to debug performance issues?"**

**Strong answer:**

> "For React-specific performance, React DevTools Profiler is often faster than the Chrome Performance panel because it speaks in React terms — components and re-renders — rather than raw JavaScript execution.
>
> I start by enabling 'Record why each component rendered' in the Profiler settings before recording. This is off by default because it adds overhead, but it's essential for performance debugging. Then I record while reproducing the slow interaction, and stop.
>
> The flame chart shows me the commit timeline — each commit is a React re-render cycle. I look for slow commits (orange/red), which are wide bars at the top. Clicking a commit shows which components rendered inside it and how long each took. Grey components were bailed out by React.memo or similar — they're efficient. Colored components rendered and I want to understand why.
>
> The crucial step: clicking a rendered component shows 'Why did this render?' at the bottom. This is exact: it says 'Props changed: { onClick: function → function }' or 'Parent re-rendered' or 'State changed: (items: Array(5) → Array(6))'. This precision tells me exactly what to fix. 'Props changed: function → function' means I need useCallback in the parent for that prop. 'Parent re-rendered' with no prop change means React.memo on the child could help — if the child is expensive.
>
> The Components panel is complementary for non-performance debugging. When I wonder 'what is the current state of this component?', I click it and see all useState values and the full props in real time. I can live-edit values to test edge cases — set a boolean to true to test an error state — without modifying source code. useDebugValue in custom hooks makes this even more useful by providing labeled context for what each hook value represents."
