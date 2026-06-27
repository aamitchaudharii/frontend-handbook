# 02 — React Interview Questions

> **"React interview questions test whether you understand React's mental model — the way it thinks about UI as a function of state — or whether you've just memorized its API. The two produce very different answers. The first explains the why behind every decision; the second recites syntax."**

This document covers the most important React interview questions for mid-to-senior positions. Questions are grouped by concept, with explanations that go beyond API description to the underlying reasoning. The goal: understand React well enough to answer questions you've never been asked before.

---

## 📚 Table of Contents

1. [React Fundamentals](#1-react-fundamentals)
2. [Hooks Deep Dive](#2-hooks-deep-dive)
3. [Performance and Optimization](#3-performance-and-optimization)
4. [State Management](#4-state-management)
5. [Component Design](#5-component-design)
6. [Advanced React](#6-advanced-react)
7. [React 18 and Concurrent Features](#7-react-18-and-concurrent-features)

---

## 1. React Fundamentals

### Q: What is the virtual DOM and how does React's reconciliation algorithm work?

**Answer:**

> The virtual DOM is a lightweight JavaScript object tree that mirrors the real DOM. When state changes, React generates a new virtual tree and diffs it against the previous one using a reconciliation algorithm. The minimum set of real DOM mutations needed to bring the DOM up to date is then applied in a single commit phase.

```
RECONCILIATION HEURISTICS (O(n) instead of optimal O(n³)):

1. Elements of different types → unmount old subtree, mount new subtree entirely
   <div> → <span>: unmount div + all children, mount span + all children
   Component A → Component B: unmount A (loses state), mount B (fresh state)

2. Same type: update existing DOM node / component with new props
   <div className="a"> → <div className="b">: only update className attribute

3. Keys in lists: O(1) lookup to match items across renders
   Without keys: compare by position → reordering causes all items to "change"
   With stable keys: match by key → only moved/added/removed items are touched
```

---

### Q: Why does React re-render? What are all the triggers?

```jsx
// React re-renders a component when:

// 1. STATE CHANGES in the component
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
// setState → component re-renders

// 2. PROPS CHANGE (parent re-renders and passes new values)
function Child({ name }) {
  return <div>{name}</div>;
}
function Parent() {
  const [name, setName] = useState("Alice");
  return <Child name={name} />;
}
// Parent re-renders → Child re-renders (even if props haven't changed, without React.memo)

// 3. CONTEXT VALUE CHANGES
const ThemeContext = createContext("light");
function Consumer() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>...</div>;
}
// When ThemeContext value changes → ALL consumers re-render

// 4. forceUpdate (class components only — avoid)
// 5. Parent re-renders (all children re-render by default)
// 6. Custom hooks that call setState internally

// IMPORTANT: re-render ≠ DOM mutation
// React reconciles on every re-render, but only applies DOM changes when needed
```

---

### Q: What is the difference between `useEffect` with different dependency arrays?

```javascript
// NO DEPENDENCY ARRAY: runs after EVERY render
useEffect(() => {
  console.log("runs on every render");
});

// EMPTY ARRAY: runs once after mount
useEffect(() => {
  console.log("runs once after mount");
  return () => console.log("cleanup on unmount");
}, []);

// WITH DEPS: runs when any dependency changes
useEffect(() => {
  console.log("runs when userId or filter changes");
  return () => console.log("cleanup before next run, and on unmount");
}, [userId, filter]);

// EXECUTION ORDER for a component's lifecycle:
// Mount:   render → commit DOM → useLayoutEffect → useEffect
// Update:  render → commit DOM → cleanup useLayoutEffect → useLayoutEffect → cleanup useEffect → useEffect
// Unmount: cleanup useLayoutEffect → cleanup useEffect
```

---

### Q: What is `useLayoutEffect`? When do you use it over `useEffect`?

```javascript
// useEffect: fires ASYNCHRONOUSLY after the browser has painted
// useLayoutEffect: fires SYNCHRONOUSLY after DOM mutations, BEFORE the browser paints

// Use useLayoutEffect when:
// 1. You need to READ from the DOM immediately after update (measure elements)
// 2. You need to MODIFY the DOM before the user sees the updated state
//    (preventing flash of incorrect content)

// Example: tooltip positioning
function Tooltip({ children, text }) {
  const tooltipRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    // Measure the tooltip BEFORE paint — user never sees it in wrong position
    const { top, left } = computeTooltipPosition(tooltipRef.current);
    setPosition({ top, left });
  }); // runs after every render, before paint

  return (
    <>
      {children}
      <div ref={tooltipRef} style={{ top: position.top, left: position.left }}>
        {text}
      </div>
    </>
  );
}

// With useEffect: tooltip flashes in wrong position, then jumps (visible to user)
// With useLayoutEffect: position computed before browser paints — no flash
```

---

## 2. Hooks Deep Dive

### Q: What rules of hooks exist and why do they exist?

```javascript
// RULE 1: Only call hooks at the top level (not in conditions, loops, nested functions)
// RULE 2: Only call hooks in React functions (components or custom hooks)

// WHY RULE 1:
// React tracks hook state by the ORDER hooks are called in each render.
// Each hook's state is stored in a linked list on the fiber.
// If hooks are called conditionally, the ORDER can change between renders.
// React would map the wrong state to the wrong hook.

// ❌ This breaks React's hook ordering guarantee:
function Component({ showExtra }) {
  const [basic, setBasic] = useState(0);
  if (showExtra) {
    const [extra, setExtra] = useState(0); // sometimes hook #2, sometimes nonexistent
  }
  useEffect(() => {}); // hook #2 or #3 depending on showExtra
}

// ✅ Condition goes INSIDE the hook, not around it:
function Component({ showExtra }) {
  const [basic, setBasic] = useState(0);
  const [extra, setExtra] = useState(0); // always hook #2
  useEffect(() => {
    if (showExtra) {
      /* logic here */
    }
  });
}
```

---

### Q: What is `useRef`? What are its two main use cases?

```javascript
// useRef returns an object { current: value } that persists across renders
// Unlike useState: updating .current does NOT trigger a re-render

// USE CASE 1: Access DOM nodes
function TextInput() {
  const inputRef = useRef(null);

  function focusInput() {
    inputRef.current.focus(); // direct DOM access
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={focusInput}>Focus</button>
    </>
  );
}

// USE CASE 2: Store mutable values that persist across renders WITHOUT triggering re-renders
function Stopwatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef(null); // stores the interval ID across renders

  function start() {
    intervalRef.current = setInterval(() => {
      setTime((t) => t + 1);
    }, 1000);
  }

  function stop() {
    clearInterval(intervalRef.current); // access the stored interval ID
  }

  return (
    <>
      <p>{time}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </>
  );
}
```

---

### Q: What is the difference between `useMemo` and `useCallback`?

```javascript
// useMemo: memoize the RESULT of a computation
const expensiveValue = useMemo(() => {
  return computeExpensiveValue(data); // re-computes only when data changes
}, [data]);

// useCallback: memoize a FUNCTION REFERENCE
const handleClick = useCallback(() => {
  processItem(id);
}, [id]); // re-creates function only when id changes

// They're equivalent: useCallback(fn, deps) = useMemo(() => fn, deps)

// WHEN TO USE EACH:
// useMemo: expensive computation that would re-run on every render without it
//          stable object reference for a React.memo'd child

// useCallback: function passed to a React.memo'd component as a prop
//              function used in a useEffect deps array
//              function stored in a ref for use in effects/intervals

// DON'T use either for cheap operations or when no one depends on reference stability
```

---

## 3. Performance and Optimization

### Q: Explain React.memo. When does it help and when does it fail?

```jsx
// React.memo: wraps a component, memoizes output based on props
// Only re-renders if any prop value has changed (shallow comparison)

const ExpensiveList = React.memo(function ExpensiveList({ items, onRemove }) {
  // Only re-renders when items or onRemove reference changes
  return items.map((item) => (
    <Item key={item.id} item={item} onRemove={onRemove} />
  ));
});

// WHEN IT HELPS:
// 1. Parent re-renders frequently (e.g., input state changing)
// 2. Child is expensive to render
// 3. Props ARE stable (primitives or memoized references)

// WHEN IT FAILS (renders anyway):
// Inline objects: props={{ theme: 'dark' }} → new object every parent render
// Inline arrows: onClose={() => setOpen(false)} → new function every parent render
// After BOTH are stable with useMemo/useCallback: memo actually works

// COMPLETE CORRECT PATTERN:
function Parent() {
  const [count, setCount] = useState(0);
  const [items, setItems] = useState(initialItems);

  // Memoize the handler so reference is stable
  const handleRemove = useCallback((id) => {
    setItems((prev) => prev.filter((i) => i.id !== id));
  }, []);

  // Memoize the config object so reference is stable
  const listConfig = useMemo(() => ({ pageSize: 20, sortBy: "name" }), []);

  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      {/* Now memo works: count changes but items/handleRemove/listConfig don't */}
      <ExpensiveList
        items={items}
        onRemove={handleRemove}
        config={listConfig}
      />
    </>
  );
}
```

---

### Q: What is "reconciliation" vs "rendering" vs "committing" in React?

```
RENDERING PHASE (pure, interruptible in React 18):
  React calls your component function(s) to produce a virtual DOM tree.
  No side effects, no DOM mutations — just computes what the output should be.
  This phase can be interrupted and restarted (Concurrent Mode).

RECONCILIATION:
  React diffs the new virtual tree against the previous one.
  Determines the minimal set of DOM mutations needed.
  Also happens during the rendering phase.

COMMIT PHASE (synchronous, non-interruptible):
  React applies the computed mutations to the real DOM.
  Runs useLayoutEffect after DOM mutations.
  This phase always completes atomically.

AFTER COMMIT:
  Browser paints the updated screen.
  React runs useEffect callbacks (asynchronously, after paint).
```

---

## 4. State Management

### Q: When would you use `useReducer` over `useState`?

```javascript
// USESTATE: simple, independent state values
const [count, setCount] = useState(0);
const [name, setName] = useState("");

// USEREDUCER: better for:
// 1. Complex state with multiple sub-values that update together
// 2. Next state depends on previous state in complex ways
// 3. State transitions with clear action semantics

const initialState = { items: [], isLoading: false, error: null };

function cartReducer(state, action) {
  switch (action.type) {
    case "ADD_ITEM":
      return {
        ...state,
        items: [...state.items, action.payload],
      };
    case "CHECKOUT_START":
      return { ...state, isLoading: true, error: null };
    case "CHECKOUT_SUCCESS":
      return { ...state, isLoading: false, items: [] };
    case "CHECKOUT_FAIL":
      return { ...state, isLoading: false, error: action.payload };
    default:
      return state;
  }
}

const [cart, dispatch] = useReducer(cartReducer, initialState);

dispatch({ type: "ADD_ITEM", payload: item });
dispatch({ type: "CHECKOUT_START" });

// ADVANTAGES:
// - State transitions are explicit and predictable
// - Reducer is a pure function → easily testable without React
// - Related state changes are atomic (no intermediate invalid states)
// - Action log tells you exactly what happened
```

---

### Q: What is the Context API? What are its limitations?

```jsx
// Context: share values down the tree without prop drilling
const ThemeContext = createContext("light");

function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

function DeepComponent() {
  const { theme } = useContext(ThemeContext); // no prop drilling needed
  return <div className={theme} />;
}

// LIMITATIONS:

// 1. PERFORMANCE: Every consumer re-renders when context value changes
//    Even if they only use one field of a multi-field context object
//    Fix: split contexts by update frequency (UserContext, ThemeContext, etc.)

// 2. NO FINE-GRAINED SUBSCRIPTIONS: can't subscribe to just one field
//    A component using context.user.name re-renders even when context.cart changes
//    Fix: separate contexts, or use a library like use-context-selector

// 3. NOT FOR HIGH-FREQUENCY UPDATES: scroll position, animation values
//    60 updates/second → 60 re-renders of all consumers
//    Fix: use refs or an observable library for high-frequency values

// 4. REFACTORING COST: adding a new context requires wrapping components
//    Fine for global providers, awkward for local shared state

// RULE OF THUMB:
// Context: theme, auth user, locale (changes infrequently, needed broadly)
// External store (Zustand/Redux): complex state, many interactions, high update frequency
```

---

## 5. Component Design

### Q: What is the difference between controlled and uncontrolled components?

```jsx
// CONTROLLED: React state is the source of truth
function ControlledInput() {
  const [value, setValue] = useState("");
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)} // every keystroke → setState → re-render
    />
  );
}

// UNCONTROLLED: DOM is the source of truth, accessed via ref
function UncontrolledInput() {
  const ref = useRef(null);
  function handleSubmit() {
    console.log(ref.current.value); // read only when needed
  }
  return (
    <>
      <input ref={ref} defaultValue="initial" /> {/* no re-renders on typing */}
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}

// USE CONTROLLED WHEN:
// - Instant validation while typing
// - Dependent fields (one field auto-fills another)
// - Programmatic value changes
// - Form wizard where values exist outside the form

// USE UNCONTROLLED WHEN:
// - Simple forms, only need values at submit
// - Performance matters (large forms, 100+ fields)
// - File inputs (always uncontrolled)
// - Integrating with non-React code
```

---

## 6. Advanced React

### Q: What are Error Boundaries? What do they NOT catch?

```jsx
// Error Boundaries catch errors during rendering, lifecycle methods, and constructors
// Must be class components (the one remaining class component use case)

class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  componentDidCatch(error, info) {
    reportError(error, info.componentStack); // log to monitoring
  }
  render() {
    return this.state.hasError ? <Fallback /> : this.props.children;
  }
}

// DOES NOT CATCH:
// - Event handlers (use try/catch inside the handler)
// - Async code (Promises, setTimeout)
// - Server-side rendering errors
// - Errors in the Error Boundary itself

// For async errors in components: use react-error-boundary's useErrorBoundary
const { showBoundary } = useErrorBoundary();
useEffect(() => {
  fetchData().catch(showBoundary); // manually send async errors to the boundary
}, []);
```

---

### Q: What is `forwardRef`? When do you need it?

```jsx
// By default: refs on custom components are not forwarded to the DOM node inside
// forwardRef: explicitly pass a ref to an inner element

// ❌ Without forwardRef: ref points to the wrapper component, not the input
const Input = ({ placeholder }) => <input placeholder={placeholder} />;
const ref = useRef();
<Input ref={ref} />; // ref.current is null (can't attach to function component)

// ✅ With forwardRef: ref points to the actual <input> DOM element
const Input = React.forwardRef(({ placeholder }, ref) => (
  <input ref={ref} placeholder={placeholder} />
));

const inputRef = useRef();
<Input ref={inputRef} placeholder="Enter value" />;
inputRef.current.focus(); // works! points to <input>

// USE CASE: design system components where consumers need direct DOM access
// Example: FancyButton, StyledInput, DatePicker where caller needs to .focus()

// COMBINE WITH useImperativeHandle for custom imperative API:
const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    select: () => inputRef.current.select(),
    clear: () => (inputRef.current.value = ""),
    // Only expose specific methods — not the full DOM node
  }));
  return <input ref={inputRef} {...props} />;
});

const ref = useRef();
<FancyInput ref={ref} />;
ref.current.clear(); // calls our custom method
```

---

## 7. React 18 and Concurrent Features

### Q: What is Concurrent Mode? What problems does it solve?

**Answer:**

> Concurrent Mode makes React rendering interruptible. In legacy mode, rendering was synchronous — once React started updating, it couldn't be paused. A large update would block the main thread for its entire duration. With Concurrent Mode, React can pause work in progress when higher-priority work arrives (like user input), handle the high-priority work, and then resume.

```jsx
// Problem it solves: large renders blocking user input
// Without Concurrent Mode: user types → React blocks rendering 500 components → keystrokes lag
// With Concurrent Mode: user types → React pauses heavy render → handles input → resumes

// startTransition: mark state updates as non-urgent
import { startTransition } from "react";

function SearchPage() {
  const [input, setInput] = useState("");
  const [results, setResults] = useState([]);

  function handleChange(e) {
    setInput(e.target.value); // urgent: show input immediately

    startTransition(() => {
      setResults(search(e.target.value)); // non-urgent: can be interrupted
    });
  }

  return (
    <>
      <input value={input} onChange={handleChange} />
      {/* Rendering 10,000 results — can be interrupted by new keystrokes */}
      <ResultsList results={results} />
    </>
  );
}

// useTransition: access isPending state
const [isPending, startTransition] = useTransition();
// isPending: true while the transition is processing (show loading indicator)
```

---

### Q: What is React Suspense? How does it work?

```jsx
// Suspense: declare loading states declaratively instead of imperatively

// HOW IT WORKS:
// A component can "suspend" by throwing a Promise during render.
// React catches the Promise, shows the fallback, and resumes when it resolves.
// The throwing component is transparently re-tried after the Promise resolves.

// With React.lazy (code splitting):
const Dashboard = React.lazy(() => import('./Dashboard'));

<Suspense fallback={<Spinner />}>
  <Dashboard /> {/* Suspends while JS chunk is loading → shows Spinner */}
</Suspense>

// With data fetching (requires suspense-compatible data source):
// TanStack Query: { suspense: true, throwOnError: true }
// React Server Components: async components suspend natively

// STREAMING SSR (React 18):
// HTML streams in as data becomes available
// Each Suspense boundary independently resolves
// User sees content progressively, not all-or-nothing

// Suspense + ErrorBoundary (complete async pattern):
<ErrorBoundary fallback={<Error />}>     {/* catches fetch errors */}
  <Suspense fallback={<Spinner />}>      {/* shows spinner while loading */}
    <UserDashboard />                    {/* fetches data, suspends during load */}
  </Suspense>
</ErrorBoundary>
```

---

### Q: What is automatic batching in React 18?

```javascript
// React 17 and earlier: state updates were batched ONLY in React event handlers
// React 18: state updates are batched EVERYWHERE

// React 17: two separate renders (in setTimeout, Promise callbacks, etc.)
setTimeout(() => {
  setCount((c) => c + 1); // render 1
  setFlag((f) => !f); // render 2
}, 1000);

// React 18: ONE render (automatic batching applies everywhere)
setTimeout(() => {
  setCount((c) => c + 1); // batched
  setFlag((f) => !f); // batched → single render
}, 1000);

// Promise callbacks, native event handlers, Timeouts: all batched in React 18

// Opt out with flushSync (rare use case):
import { flushSync } from "react-dom";
flushSync(() => setCount((c) => c + 1)); // forces immediate render
// DOM is updated synchronously here — useful before measuring DOM dimensions
flushSync(() => setFlag((f) => !f)); // another immediate render
```

---

## Quick-Reference: Common "Gotcha" Questions

```javascript
// Q: What happens if you call setState in the render function?
function Bad() {
  const [count, setCount] = useState(0);
  setCount(count + 1); // ❌ called during render → infinite re-render loop!
  return <div>{count}</div>;
}

// Q: Can you call hooks conditionally?
function Conditional({ show }) {
  if (show) {
    const [x] = useState(0); // ❌ violates Rules of Hooks
  }
}

// Q: Why does this not update?
const [arr, setArr] = useState([1, 2, 3]);
arr.push(4);
setArr(arr); // ❌ same reference! React won't see a change
setArr([...arr]); // ✅ new array reference

// Q: When does useEffect's cleanup run?
// Before the effect runs again (with changed deps)
// AND when the component unmounts

// Q: What is the key prop for?
// Tell React's reconciler how to match list items across renders
// Stable unique keys → only actually changed items update
// Index as key → any reorder makes all items appear "changed"
```
