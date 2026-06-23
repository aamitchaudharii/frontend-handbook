# 03 — Premature Optimization

> **"Premature optimization is the root of all evil — or at least most of it — in programming. The discipline of measuring before optimizing is not about being lazy; it's about spending finite engineering time on problems that actually exist. Every optimization you make before measuring is a bet. Most bets lose."**
> — Donald Knuth (paraphrased)

Premature optimization in React is the practice of applying performance techniques before measuring whether a performance problem actually exists. `React.memo` on every component, `useMemo` on every derived value, `useCallback` on every function — these feel like good hygiene but are often noise that makes code harder to read, introduces bugs through stale closures, and provides zero measurable benefit. This document covers the most common premature optimization patterns, how to recognize them, when optimization is genuinely warranted, and how to measure before acting.

---

## 📚 Table of Contents

1. [Why Premature Optimization Is Harmful](#1-why-premature-optimization-is-harmful)
2. [The Overuse of React.memo](#2-the-overuse-of-reactmemo)
3. [The Overuse of useMemo](#3-the-overuse-of-usememo)
4. [The Overuse of useCallback](#4-the-overuse-of-usecallback)
5. [Premature Code Splitting](#5-premature-code-splitting)
6. [Premature State Normalization](#6-premature-state-normalization)
7. [Premature Virtualization](#7-premature-virtualization)
8. [When Optimization IS Warranted](#8-when-optimization-is-warranted)
9. [How to Measure First](#9-how-to-measure-first)
10. [Good Practices](#10-good-practices)
11. [Bad Practices](#11-bad-practices)
12. [Common Mistakes](#12-common-mistakes)
13. [Interview-Level Explanation](#13-interview-level-explanation)
14. [Exercises](#14-exercises)

---

## 1. Why Premature Optimization Is Harmful

```
THE COSTS OF UNNECESSARY OPTIMIZATION:

Readability:
  const value = useMemo(() => a + b, [a, b]);
  // A reader must ask: "is this expensive? why is this memoized?"
  // Answer: probably isn't. useMemo obscures simple arithmetic.

Maintenance:
  const handleClick = useCallback(() => doSomething(id), [id]);
  // Every future change to the callback must check if deps are correct
  // Stale closures → bugs if deps array is wrong
  // No memoization needed: function is not passed to React.memo'd children

Bug surface:
  const processedData = useMemo(() => transform(data), [data]);
  // If `transform` has hidden dependencies not in the deps array:
  // silent staleness bug that's hard to reproduce

Engineering time:
  Time spent adding useMemo everywhere = time NOT spent on features, tests, or
  actual performance problems that users are experiencing

THE ACTUAL COST OF A RE-RENDER:
  A React re-render is typically 0.01ms to 2ms for small components.
  If a component renders 10 extra times per second: 10ms × 10 = 100ms wasted.
  Visible threshold: 50ms+ of jank.
  Conclusion: most "unnecessary" renders are invisible to users.

  When it matters: large lists, expensive computations, high-frequency events.
  For most components: re-renders are cheap enough to not warrant optimization.
```

---

## 2. The Overuse of React.memo

```jsx
// ❌ React.memo on a component whose parent never re-renders unnecessarily
const UserProfile = React.memo(function UserProfile({ user }) {
  return <div>{user.name}</div>;
});

// React.memo adds overhead:
// 1. Shallow prop comparison on EVERY render of the parent
// 2. If props are new references (objects, arrays) from parent: memo fails anyway
// 3. Component is wrapped in an extra layer — harder to see in DevTools

// React.memo is USEFUL when:
// - The component is provably expensive to render (large subtree or complex computation)
// - The parent re-renders frequently (e.g., on every keystroke)
// - The component's props are stable (primitives or memoized references)
// ALL THREE conditions should hold; hitting one out of three usually doesn't help

// ✅ When React.memo genuinely helps:
const ExpensiveChart = React.memo(function ExpensiveChart({ data, options }) {
  // Renders 200 SVG elements — takes ~20ms
  // Used inside a form that re-renders on every keystroke
  // data and options are memoized with useMemo by the parent
  return <SVGChart data={data} options={options} />;
});
// This is worth it: expensive render + frequent parent updates + stable props

// ❌ React.memo when parent renders infrequently:
const Header = React.memo(function Header({ title }) {
  return <h1>{title}</h1>;
});
// Header renders once per page navigation — React.memo saves ~0ms and costs readability
```

### React.memo Fails Silently on Object Props

```jsx
// ❌ React.memo provides zero benefit when parent passes new objects
const MemoizedCard = React.memo(function Card({ config }) {
  /* ... */
});

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
      <MemoizedCard config={{ theme: "dark", size: "large" }} />{" "}
      {/* new object every render! */}
    </>
  );
}
// config is { theme: 'dark', size: 'large' } — new object reference on every render
// React.memo: compares oldConfig !== newConfig (different references) → ALWAYS re-renders
// The memo provides ZERO benefit — Parent must also memoize the config prop:

const config = useMemo(() => ({ theme: "dark", size: "large" }), []);
// Now memo works — but you've added useMemo to the parent too, increasing complexity
```

---

## 3. The Overuse of useMemo

```jsx
// ❌ useMemo on cheap operations (adds overhead without benefit)
function Component({ a, b }) {
  const sum = useMemo(() => a + b, [a, b]); // single addition
  const label = useMemo(() => `${a} + ${b} = ${sum}`, [a, b, sum]); // string concat
  const isLong = useMemo(() => label.length > 20, [label]); // comparison
  // ...
}

// The cost of useMemo itself:
// - Stores the deps array in fiber memory
// - Shallow-compares deps on every render (O(n) comparison)
// - Closure allocation
// For simple arithmetic/string ops: useMemo overhead > computation cost

// ❌ useMemo with no downstream consumer needing reference stability
function Component({ items }) {
  const processed = useMemo(
    () =>
      items
        .filter((i) => i.active)
        .sort((a, b) => a.name.localeCompare(b.name)),
    [items],
  );
  // processed is passed to a non-memoized component
  // That component will re-render anyway when its parent re-renders
  // useMemo prevents "expensive" sort but the expensive part is the re-render itself
}

// ✅ When useMemo is justified:
function DataGrid({ rows }) {
  const sortedRows = useMemo(() => {
    // 10,000 rows, complex multi-key sort: ~50ms
    return rows.slice().sort(complexComparator);
  }, [rows]);
  // This is expensive AND the parent re-renders on unrelated state changes
  // Without useMemo: 50ms computation on every parent re-render
  // WITH useMemo: 50ms only when rows actually change
  return <Grid rows={sortedRows} />;
}
```

### The useMemo Dependency Array Trap

```jsx
// ❌ useMemo with unstable dependencies defeats itself
function Component({ user, onUpdate }) {
  const config = useMemo(
    () => ({
      userId: user.id,
      callback: onUpdate, // onUpdate is a new function every parent render
    }),
    [user, onUpdate],
  ); // onUpdate changes every render → memo never hits!
  // useMemo runs on every render anyway — adds overhead, zero benefit
}

// ✅ Fix: stabilize dependencies first
function Parent() {
  const handleUpdate = useCallback(() => {
    /* ... */
  }, []); // stable reference
  return <Component user={user} onUpdate={handleUpdate} />;
}
```

---

## 4. The Overuse of useCallback

```jsx
// ❌ useCallback when the function is only called inline, never passed to memo'd children
function Component({ id }) {
  const handleClick = useCallback(() => {
    console.log(id);
  }, [id]);

  return <button onClick={handleClick}>Click</button>;
  // This button is a plain <button>, not a React.memo'd component
  // The inline arrow function () => console.log(id) costs the same as useCallback
  // useCallback adds: deps array storage + comparison overhead
  // Use: saves nothing — <button> doesn't care about reference stability
}

// ❌ useCallback "just in case" for future memoization
const handleClose = useCallback(() => setIsOpen(false), []);
// If nothing currently depends on reference stability: no benefit today
// And if something will need it tomorrow: add it then, not now

// ✅ When useCallback is justified:
const MemoizedList = React.memo(function List({ items, onRemove }) {
  return items.map((item) => (
    <Item key={item.id} item={item} onRemove={onRemove} />
  ));
});

function Parent({ items }) {
  // ✅ onRemove is passed to React.memo'd component — reference stability matters
  const handleRemove = useCallback((id) => {
    setItems((prev) => prev.filter((i) => i.id !== id));
  }, []); // stable: no deps needed because setItems updater form used

  return <MemoizedList items={items} onRemove={handleRemove} />;
}
// Without useCallback: new function reference every render → MemoizedList always re-renders
// With useCallback: stable reference → MemoizedList only re-renders when items changes
```

---

## 5. Premature Code Splitting

```javascript
// ❌ Splitting every component into its own lazy-loaded chunk
const Button = React.lazy(() => import("./Button"));
const Input = React.lazy(() => import("./Input"));
const Checkbox = React.lazy(() => import("./Checkbox"));
const Label = React.lazy(() => import("./Label"));
// Each of these adds:
// - A Suspense boundary requirement
// - An async module evaluation overhead
// - Potential waterfall: parent chunk loads → discovers child imports → loads those
// For tiny components: the overhead vastly exceeds the benefit

// Code splitting pays off when:
// - The chunk is large (> 20KB minified+gzipped)
// - The chunk is NOT needed for the initial page render
// - The chunk is used by only some users (e.g., admin panel)

// ✅ Split at meaningful boundaries:
const AdminPanel = React.lazy(() => import("./features/admin/AdminPanel"));
const RichTextEditor = React.lazy(
  () => import("./features/editor/RichTextEditor"),
);
const AnalyticsDashboard = React.lazy(
  () => import("./features/analytics/Dashboard"),
);
// Each of these is large, loaded conditionally, and benefits from the split
```

---

## 6. Premature State Normalization

```javascript
// ❌ Normalizing simple data before it proves necessary
// From a 20-item list:
const state = {
  entities: {
    products: {
      byId: {
        1: { id: "1", name: "Product A" },
        2: { id: "2", name: "Product B" },
      },
      allIds: ["1", "2"],
    },
  },
};
// For 20 items: flat array is simpler, faster for most operations,
// and doesn't need a normalization library

// ✅ Use normalization when:
// - 1000+ entities where byId lookup is performance-critical
// - Entities are referenced from multiple places and must stay in sync
// - Real update performance problems have been measured

// For small to medium data: keep it simple
const state = {
  products: [
    { id: "1", name: "Product A" },
    { id: "2", name: "Product B" },
  ],
};
```

---

## 7. Premature Virtualization

```jsx
// ❌ Virtualizing a 50-item list "just in case"
import { FixedSizeList } from "react-window";

function ProductList({ products }) {
  // products.length = 50
  return (
    <FixedSizeList
      height={400}
      itemCount={products.length}
      itemSize={80}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <ProductCard product={products[index]} />
        </div>
      )}
    </FixedSizeList>
  );
}
// Costs: fixed item heights required, more complex code, harder to customize
// For 50 items: 50 DOM nodes is completely fine — no virtualization needed

// ✅ Virtualize when:
// - 500+ items AND scroll performance is measured to be poor
// - Items are complex enough that 500 DOM nodes causes frame drops
// - Profiler shows actual scroll-related jank
```

---

## 8. When Optimization IS Warranted

```
SIGNALS THAT OPTIMIZATION IS NEEDED (not speculation — measured problems):

1. USER-REPORTED PERFORMANCE ISSUE:
   "The filter dropdown is slow when I type" → measure → find cause → fix

2. PROFILER SHOWS > 50ms RENDERS:
   React DevTools Profiler: "Component X took 80ms to render" → investigate

3. LONG TASK IN PERFORMANCE PANEL:
   Chrome DevTools: long task blocking input for 200ms → find the cause

4. INP FAILING (> 200ms):
   Web Vitals: INP 350ms → profile interactions → find the expensive component

5. MEASURED EXPENSIVE COMPUTATION:
   console.time() shows transform(data) taking 80ms → useMemo is warranted

FOR GENUINE PERFORMANCE PROBLEMS:
  React.memo:     expensive component + frequent parent renders + stable props
  useMemo:        expensive computation (> 1ms) + runs on unrelated re-renders
  useCallback:    function passed to React.memo'd component as a prop
  Code splitting: large chunk (> 20KB) not needed on initial load
  Virtualization: 500+ items with measured scroll jank
```

---

## 9. How to Measure First

### React DevTools Profiler

```
React DevTools → Profiler tab → Record → interact → Stop

Look for:
  "Why did this render?" → shows which prop/state changed
  Render duration: is it > 16ms? (would drop frames at 60fps)
  Commit bar chart: which commits were expensive?

The profiler tells you WHERE the problem is before you guess.
```

### Chrome Performance Panel

```javascript
// Mark custom performance entries around suspected slow code
performance.mark("transform-start");
const result = expensiveTransform(data);
performance.mark("transform-end");
performance.measure("transform", "transform-start", "transform-end");

// View in Performance panel → User Timing section
// See exactly how long the transform took, in context of the full frame
```

### Simple Timing

```javascript
// Quick check before adding useMemo
const start = performance.now();
const result = transform(data);
console.log(`transform took ${performance.now() - start}ms`);

// If < 1ms: useMemo adds more overhead than it saves
// If > 5ms: useMemo may be warranted (evaluate how often this runs)
// If > 16ms: definitely optimize (drops frames)
```

---

## 10. Good Practices

### ✅ Write readable code first, optimize when measured

```jsx
// ✅ Simple, readable — optimize later if profiling shows a problem
function FilteredList({ items, query }) {
  const filtered = items.filter((i) => i.name.includes(query));
  return filtered.map((item) => <Item key={item.id} item={item} />);
}

// Only add useMemo when:
// - items.length is large (1000+)
// - This component re-renders frequently (on every keystroke)
// - Profiler shows this filter is taking meaningful time
```

### ✅ Measure with the Profiler before adding memo, useMemo, useCallback

```
Team rule: "No performance optimization without a performance measurement."
Every useMemo should have a comment explaining WHY it was added.
// useMemo: items.sort takes 40ms on 10k items (profiled, Chrome, mid-range device)
```

### ✅ When adding React.memo, verify it actually helps

```javascript
// Verify memo is working: if the component still re-renders, memo isn't helping
if (process.env.NODE_ENV === "development") {
  const prevPropsRef = useRef({});
  useEffect(() => {
    const prev = prevPropsRef.current;
    const changed = Object.keys(props).filter((k) => props[k] !== prev[k]);
    if (changed.length) {
      console.log(`${displayName} re-rendered, changed props:`, changed);
    }
    prevPropsRef.current = props;
  });
}
// If you see "re-rendered" every time → check why the prop reference is unstable
```

---

## 11. Bad Practices

### ❌ useMemo/useCallback in every component "for performance"

```jsx
// ❌ This is defensive coding, not engineering
function Component({ user, onClick }) {
  const userName = useMemo(() => user.firstName + " " + user.lastName, [user]); // string concat!
  const handleClick = useCallback(() => onClick(user.id), [onClick, user.id]);
  const userInitials = useMemo(
    () => userName.slice(0, 2).toUpperCase(),
    [userName],
  );
  return <div onClick={handleClick}>{userInitials}</div>;
}
// All three are pointless: cheap operations, no memo'd consumers
```

### ❌ React.memo as the default for ALL components

```jsx
// ❌ Team rule: "all components should be wrapped in React.memo"
export default React.memo(Button);
export default React.memo(Text);
export default React.memo(Icon);
// This is cargo-cult optimization: adds overhead everywhere, helps nowhere specific
```

---

## 12. Common Mistakes

### Mistake 1 — Thinking React.memo prevents re-renders unconditionally

```jsx
// ❌ React.memo does a SHALLOW comparison — objects/arrays still fail
const Memoized = React.memo(Component);

// Parent:
<Memoized options={{ a: 1 }} />  // new object every render → memo fails
<Memoized items={[...list]} />   // new array every render → memo fails
<Memoized style={{ color: 'red' }} />  // new object → memo fails

// memo only helps when props are:
// - Primitives (strings, numbers, booleans)
// - SAME reference (memoized objects/arrays, stable function refs)
```

### Mistake 2 — Adding useMemo to fix stale closure bugs instead of fixing deps

```jsx
// ❌ Using useMemo to avoid a stale closure problem instead of fixing deps
const result = useMemo(() => compute(latestValue), []); // empty deps → stale!
// This "memoizes" the stale value — that's a bug, not an optimization

// ✅ Fix the deps, not the optimization:
const result = useMemo(() => compute(latestValue), [latestValue]);
```

### Mistake 3 — Optimizing before the feature is even correct

```jsx
// ❌ Adding performance optimizations to code that isn't correct yet
function ProductSearch({ query }) {
  const results = useMemo(
    () => products.filter((p) => fuzzyMatch(p.name, query)), // fuzzyMatch has bugs!
    [query],
  );
  // The feature doesn't work correctly — optimizing it first makes debugging harder
  // and means the optimization will be thrown away when the logic changes
}

// ✅ Make it work correctly first. Optimize second, if needed.
function ProductSearch({ query }) {
  const results = products.filter((p) => fuzzyMatch(p.name, query)); // simple, debuggable
  // Once this is correct and tested, add useMemo IF profiling shows it's slow
}
```

---

## 13. Interview-Level Explanation

> **"How do you approach performance optimization in React? When would you use useMemo and useCallback?"**

**Strong answer:**

> "My approach is measure first, optimize second. Every optimization I make before measuring is a guess, and guesses in performance work are wrong often enough to cost more than they save. Before adding any memoization, I want to see a specific, measured problem: the React DevTools Profiler showing a component taking 80ms to render, the Chrome Performance panel showing a long task causing input delay, or a Web Vitals report showing INP above 200ms.
>
> For `useMemo`: the right question is whether the computation is genuinely expensive AND runs on renders that don't change its inputs. A quick sanity check is `performance.now()` before and after — if it's under 1ms, `useMemo` is adding overhead without benefit. For most derived state — filtering a list of 50 items, concatenating strings, building a URL — the computation is trivially cheap and useMemo makes the code harder to read for no gain. The genuinely useful cases are things like sorting 10,000 records, building a search index, or running a complex transform where I've measured the cost.
>
> For `useCallback`: the only reason to stabilize a function reference is if that reference is passed to a `React.memo`'d child, used in a `useEffect` dependency array, or passed to a virtualized list that needs referential stability. Wrapping a function that's only used inline in JSX event handlers in `useCallback` does nothing — the `<button>` element doesn't care if the onClick reference is the same between renders.
>
> For `React.memo`: it only works when the props are stable references. If a parent passes a new object literal or a new arrow function on every render, memo's shallow comparison will always find them unequal and re-render anyway. You need both the memo on the child AND stable references from the parent, which often requires `useMemo` and `useCallback` in the parent as well. This complexity is only worth the effort when the child is genuinely expensive to render and the parent genuinely re-renders frequently.
>
> The deeper issue with premature optimization is that it makes code harder to reason about. Stale closure bugs from incorrect dependency arrays, 'why is this memoized?' questions from readers, and incorrect optimization intuitions about which parts are actually slow all come from optimizing before measuring. Write readable code first. Measure. Then apply targeted optimizations to the specific bottleneck you found."

---

## 14. Exercises

### Exercise 1 — Audit this component for premature optimizations

```jsx
function UserDashboard({ userId }) {
  const [data, setData] = useState(null);

  const fetchUser = useCallback(async () => {
    const user = await api.getUser(userId);
    setData(user);
  }, [userId]);

  useEffect(() => {
    fetchUser();
  }, [fetchUser]);

  const greeting = useMemo(() => `Hello, ${data?.name}!`, [data?.name]);
  const initial = useMemo(() => data?.name?.[0]?.toUpperCase(), [data?.name]);
  const isAdmin = useMemo(() => data?.role === "admin", [data?.role]);

  const handleRefresh = useCallback(() => fetchUser(), [fetchUser]);

  if (!data) return <Spinner />;

  return (
    <div>
      <h1>{greeting}</h1>
      <Avatar initial={initial} />
      {isAdmin && <AdminBadge />}
      <button onClick={handleRefresh}>Refresh</button>
    </div>
  );
}
```

<details>
<summary>Solution</summary>

```
PREMATURE OPTIMIZATIONS FOUND:

1. useCallback for fetchUser:
   fetchUser is only used in one useEffect — it's not passed to any memo'd component.
   The useCallback + useEffect[fetchUser] pattern adds complexity for no benefit.
   The useEffect depends on userId directly:

   useEffect(() => {
     api.getUser(userId).then(setData);
   }, [userId]);
   // No useCallback needed — simpler, same behavior

2. useMemo for greeting (string template literal):
   `Hello, ${data?.name}!` — string concatenation is nanoseconds.
   useMemo overhead > operation cost.

3. useMemo for initial (single character access + toUpperCase):
   data?.name?.[0]?.toUpperCase() — trivial operation.

4. useMemo for isAdmin (boolean comparison):
   data?.role === 'admin' — single comparison: essentially free.

5. useCallback for handleRefresh:
   handleRefresh is passed to a plain <button> — not a React.memo'd component.
   <button> has no use for reference stability.
   A simple () => api.getUser(userId).then(setData) inline is fine.

CLEANED UP VERSION:
function UserDashboard({ userId }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    api.getUser(userId).then(setData);
  }, [userId]);

  // Derived values — no memoization needed (trivially cheap)
  const greeting = `Hello, ${data?.name}!`;
  const initial  = data?.name?.[0]?.toUpperCase();
  const isAdmin  = data?.role === 'admin';

  if (!data) return <Spinner />;

  return (
    <div>
      <h1>{greeting}</h1>
      <Avatar initial={initial} />
      {isAdmin && <AdminBadge />}
      <button onClick={() => api.getUser(userId).then(setData)}>Refresh</button>
    </div>
  );
}

WHEN TO ADD OPTIMIZATION BACK:
  - If this component is inside a parent that re-renders on every keystroke (high frequency)
  - AND profiling shows UserDashboard itself is taking > 16ms
  - THEN consider targeted optimization, starting with the most expensive operation

None of the removed optimizations would provide measurable benefit in practice.
```

</details>

---

## 🔗 Related Topics

- [`performance/07-memoization.md`](../performance/07-memoization.md) — When memoization IS the right tool
- [`rendering/02-virtual-dom.md`](../rendering/02-virtual-dom.md) — What re-renders actually cost
- [`browser-internals/01-rendering-pipeline.md`](../browser-internals/01-rendering-pipeline.md) — Full rendering pipeline

---

<div align="center">

**Next:** [`anti-patterns/04-stale-closures.md`](./04-stale-closures.md) →

</div>
