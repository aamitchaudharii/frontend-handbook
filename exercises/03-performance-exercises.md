# 03 — Performance Exercises

> **"Performance exercises are different from correctness exercises — there's rarely one right answer, only better and worse tradeoffs given specific constraints. The skill being practiced is profiling first, then applying the narrowest fix that solves the measured problem."**

These exercises present realistic performance scenarios: diagnose the bottleneck, then apply the fix. Each includes the symptom, the diagnostic approach, and the solution with reasoning about why it works and what the tradeoffs are.

---

## Diagnosing Re-renders

### Exercise 1.1 — Find the Unnecessary Re-render

```jsx
// This Dashboard re-renders ALL widgets whenever the search input changes,
// even though most widgets don't depend on the search query.
// Diagnose the issue and fix it.

function Dashboard() {
  const [searchQuery, setSearchQuery] = useState("");
  const [user, setUser] = useState(null);
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    fetchUser().then(setUser);
    fetchNotifications().then(setNotifications);
  }, []);

  return (
    <div>
      <SearchBar value={searchQuery} onChange={setSearchQuery} />
      <UserProfile user={user} />
      <NotificationPanel notifications={notifications} />
      <RevenueChart />
      <ActivityFeed />
    </div>
  );
}
```

<details>
<summary>Solution</summary>

```jsx
// DIAGNOSIS:
// Dashboard owns searchQuery state. Every keystroke updates searchQuery,
// causing Dashboard to re-render, which re-renders ALL its children —
// UserProfile, NotificationPanel, RevenueChart, ActivityFeed — none of
// which depend on searchQuery at all.

// React DevTools Profiler would show: "Parent re-rendered" for all four
// widgets on every keystroke.

// FIX: Move search state down to the component that actually owns it —
// or isolate it so its re-renders don't cascade to unrelated siblings.

// Option A: Extract SearchBar + its state into its own component
function SearchSection({ onSearch }) {
  const [searchQuery, setSearchQuery] = useState("");

  function handleChange(value) {
    setSearchQuery(value);
    onSearch(value); // notify parent only when needed (e.g., debounced)
  }

  return <SearchBar value={searchQuery} onChange={handleChange} />;
}

function Dashboard() {
  const [user, setUser] = useState(null);
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    fetchUser().then(setUser);
    fetchNotifications().then(setNotifications);
  }, []);

  return (
    <div>
      <SearchSection onSearch={(q) => console.log("search:", q)} />
      <UserProfile user={user} />
      <NotificationPanel notifications={notifications} />
      <RevenueChart />
      <ActivityFeed />
    </div>
  );
}

// Now: typing in the search box only re-renders SearchSection.
// Dashboard's other children are unaffected because Dashboard itself
// doesn't re-render — searchQuery state no longer lives there.

// Option B (if Dashboard genuinely needs searchQuery for filtering other
// data): wrap the unrelated widgets in React.memo so they bail out
// when their own props haven't changed:
const UserProfile = React.memo(function UserProfile({ user }) {
  /* ... */
});
const RevenueChart = React.memo(function RevenueChart() {
  /* ... */
});
// This works but Option A is cleaner when the state truly doesn't need
// to live at the Dashboard level.
```

</details>

---

### Exercise 1.2 — Diagnose Using the Profiler

```
SCENARIO: A user reports that typing in a comment box feels laggy —
noticeable input delay after each keystroke.

Walk through how you would diagnose this using React DevTools and
Chrome DevTools, step by step. What would you look for at each stage?
```

<details>
<summary>Solution</summary>

```
DIAGNOSTIC WALKTHROUGH:

STEP 1: Confirm and quantify
  Chrome DevTools → Performance → Record → type several characters → Stop
  Look at the FPS meter and Long Task indicators (red hatching)
  If typing causes Long Tasks (>50ms): confirmed performance issue, not perception

STEP 2: Identify what's running during the lag
  Zoom into the Long Task in the flame chart
  Look at the widest bar inside it — this is the actual bottleneck
  Common findings:
    - Wide "Recalculate Style" / "Layout" bars → DOM thrashing
    - Wide yellow (JS) bar → expensive JavaScript execution
    - Many repeated small bars → re-render of many components

STEP 3: Switch to React DevTools Profiler for React-specific detail
  Enable "Record why each component rendered" in Profiler settings
  Record → type a character → Stop
  Look at the commit — how many components rendered for ONE keystroke?
  If many unrelated components rendered: this points to state living too high
  (see Exercise 1.1 pattern) or a Context update affecting many consumers

STEP 4: Click the most expensive component in the commit
  Check "Why did this render?"
  If "Parent re-rendered": the issue might be upstream — keep climbing
    the tree to find where state actually changed
  If "Props changed: { onChange: function → function }": an inline
    function reference is breaking memoization somewhere

STEP 5: Check for expensive computation in the render path
  Search the component tree for: filter(), sort(), map() with expensive
  callbacks, or non-memoized derived values that run on every keystroke
  Add console.time/console.timeEnd around suspected expensive operations
  if (operation takes > 5ms): candidate for useMemo

STEP 6: Apply the narrowest fix
  If state lives too high: move it down (colocate) or extract a sub-component
  If unrelated siblings re-render: React.memo + stabilize their props
  If expensive computation: useMemo with measured justification
  If layout thrashing: batch DOM reads/writes (see rendering/01-dom-batching.md)

STEP 7: Re-measure to confirm the fix worked
  Re-run the Performance recording
  Confirm: Long Tasks during typing are gone or significantly reduced
  Re-run React Profiler: confirm only the expected components render now
```

</details>

---

## Memoization Decisions

### Exercise 2.1 — Should You Memoize This?

```jsx
// For each of the following, decide: should this use useMemo/useCallback?
// Justify your answer.

// A.
function Component({ items }) {
  const total = useMemo(
    () => items.reduce((sum, i) => sum + i.price, 0),
    [items],
  );
  return <div>{total}</div>;
}

// B.
function Component({ onSave }) {
  const handleClick = useCallback(() => onSave(), [onSave]);
  return <button onClick={handleClick}>Save</button>;
}

// C.
function Component({ rows }) {
  const sorted = useMemo(
    () => [...rows].sort((a, b) => complexMultiKeyCompare(a, b)),
    [rows],
  );
  return <DataGrid rows={sorted} />; // DataGrid is NOT memoized
}

// D.
const ExpensiveChart = React.memo(function ExpensiveChart({ data, theme }) {
  // renders 500 SVG elements, ~25ms render time
  return <SVGChart data={data} theme={theme} />;
});

function Dashboard() {
  const [count, setCount] = useState(0); // updates frequently (e.g., live counter)
  const chartData = useMemo(() => processChartData(rawData), [rawData]);
  const themeConfig = useMemo(() => ({ color: "blue", size: "lg" }), []);

  return (
    <>
      <span>{count}</span>
      <ExpensiveChart data={chartData} theme={themeConfig} />
    </>
  );
}
```

<details>
<summary>Solution</summary>

```
A. NOT JUSTIFIED (most cases)
   items.reduce() summing prices is O(n) but extremely cheap per item —
   for typical list sizes (under a few thousand items), this runs in
   microseconds. useMemo overhead (deps comparison, cache storage) likely
   costs more than the computation itself.
   VERDICT: Remove useMemo unless `items` has 10,000+ elements AND this
   component re-renders frequently for unrelated reasons. Measure first.

B. NOT JUSTIFIED (as shown)
   handleClick is passed to a plain <button>, not a React.memo'd component.
   <button> doesn't care about function reference stability — a new
   function on every render costs nothing meaningful here.
   VERDICT: Remove useCallback. Only add it back if handleClick is later
   passed to a memoized child or used in a dependency array.

C. JUSTIFIED — sort() is genuinely expensive, but check the second half
   complexMultiKeyCompare suggests a non-trivial comparator; sorting is
   O(n log n) and could be meaningfully slow for large datasets.
   useMemo is correctly applied to avoid re-sorting on every render of
   the parent for unrelated reasons.
   HOWEVER: DataGrid is not memoized — so even with sorted memoized,
   DataGrid still re-renders whenever Component re-renders (just doesn't
   re-sort). If DataGrid itself is expensive to render: wrap it in
   React.memo too, otherwise the useMemo only solves half the problem.

D. JUSTIFIED — textbook case for full optimization
   ExpensiveChart genuinely takes 25ms to render (measured).
   Dashboard re-renders frequently because of `count` (a live counter,
   implying frequent updates).
   Both `chartData` and `themeConfig` are memoized, giving ExpensiveChart
   STABLE references even when Dashboard re-renders for the `count` update.
   React.memo on ExpensiveChart means: when count changes, ExpensiveChart's
   props (chartData, theme) are unchanged references → memo bails out →
   ExpensiveChart does NOT re-render → the 25ms render cost is avoided
   on every count tick.
   This is the complete, correctly-applied optimization pattern.

GENERAL PRINCIPLE:
  Memoization needs three things aligned to be worthwhile:
  1. The computation/render is measurably expensive
  2. The component re-renders for reasons UNRELATED to this value
  3. ALL the pieces (memo + memoized inputs) are in place — partial
     application (just useMemo without React.memo on the consumer, or
     vice versa) often provides no benefit at all.
```

</details>

---

## Bundle and Loading Performance

### Exercise 3.1 — Reduce Bundle Size

```
SCENARIO: Your production bundle is 850KB gzipped. The performance budget
is 250KB for the initial load. You run `webpack-bundle-analyzer` and find:

- moment.js with all locales: 230KB
- lodash (full import): 70KB
- A chart library used only on the /analytics page: 180KB
- Your application code: 200KB
- A UI component library (fully imported): 120KB
- Other (polyfills, runtime, etc.): 50KB

Propose a plan to get under the 250KB budget for the initial load.
```

<details>
<summary>Solution</summary>

```
ANALYSIS AND FIXES:

1. moment.js with all locales (230KB) → BIGGEST WIN
   Fix: Switch to date-fns (tree-shakeable, ~2-5KB for typically-used functions)
        or dayjs (2KB total, moment-compatible API)
   If moment must stay: use moment-locales-webpack-plugin to strip unused locales
   Expected savings: ~220KB

2. lodash full import (70KB) → EASY WIN
   Problem: `import _ from 'lodash'` imports the ENTIRE library
   Fix: import only what's used: `import debounce from 'lodash/debounce'`
        or switch to lodash-es with proper tree-shaking
        or replace simple utilities with native JS (many lodash functions
        have native equivalents: _.map → Array.map, etc.)
   Expected savings: ~55-65KB (depending on how much is actually used)

3. Chart library used only on /analytics (180KB) → CODE SPLIT
   Problem: loaded on EVERY page even though only /analytics needs it
   Fix: React.lazy() + dynamic import, loaded only when /analytics route
        is visited
   const AnalyticsPage = React.lazy(() => import('./pages/AnalyticsPage'));
   Expected savings: 180KB removed from initial bundle (still loaded when needed)

4. UI component library fully imported (120KB) → PARTIAL WIN
   Problem: likely importing the whole library instead of used components
   Fix: check if the library supports tree-shaking (ESM + sideEffects: false
        in package.json) — if so, named imports should already tree-shake
        If not: switch to per-component imports:
        import Button from 'ui-library/Button' instead of
        import { Button } from 'ui-library'
   Expected savings: depends on usage — if using 30% of components, ~80KB savings

5. Application code (200KB) → ROUTE-LEVEL SPLITTING
   Fix: split each route into its own chunk via React.lazy()
        Only the current route's code loads initially
   Expected savings: depends on route count — if 5 roughly-equal routes,
   initial bundle gets ~160KB of this (only the home/landing route)

CALCULATION AFTER FIXES:
  moment → date-fns:        230KB → 10KB   (-220KB)
  lodash → per-function:     70KB → 10KB   (-60KB)
  chart library:            180KB → 0KB    (-180KB, lazy loaded)
  UI library → tree-shaken: 120KB → 40KB   (-80KB)
  app code → route split:   200KB → 80KB   (-120KB, other routes lazy)
  other (unchanged):         50KB → 50KB

  NEW TOTAL: 10 + 10 + 0 + 40 + 80 + 50 = 190KB
  ✅ Under the 250KB budget, with room to spare

VERIFICATION:
  Re-run webpack-bundle-analyzer after changes
  Run Lighthouse to confirm bundle size improvement translates to
  faster FCP/LCP/TTI in practice
  Set up a CI budget check (e.g., bundlesize or size-limit) to prevent
  regression
```

</details>

---

## Network and Loading Strategy

### Exercise 4.1 — Fix the Waterfall

```
SCENARIO: A product page takes 4 seconds to show content. The Network
panel waterfall shows:

1. index.html: 0-200ms
2. main.js (bundle): 200-900ms (loads after HTML parses)
3. App makes API call for product data: 900-1900ms (starts after JS loads)
4. App makes API call for reviews: 1900-2700ms (starts AFTER product data resolves)
5. App makes API call for related products: 2700-3500ms (starts after reviews)
6. Final render: 3500-4000ms

Identify the waterfall problems and propose fixes.
```

<details>
<summary>Solution</summary>

```
PROBLEMS IDENTIFIED:

PROBLEM 1: API calls don't start until JS bundle loads (900ms wasted)
  The browser COULD start the product data fetch as soon as it knows the
  product ID (which might be available from the URL, not requiring JS)

PROBLEM 2: Sequential API calls that should be parallel
  Reviews and related products don't depend on each other OR on the
  product data response. They're being fetched sequentially for no reason.
  This adds (2700-1900) + (3500-2700) = 1600ms of unnecessary serial waiting

FIXES:

FIX 1: Preload critical API call before JS executes
<link rel="preconnect" href="https://api.example.com">
<!-- Or even better: trigger the fetch immediately in an inline script -->
<script>
  window.__productDataPromise = fetch(`/api/products/${productId}`).then(r => r.json());
</script>
<!-- React app later does: -->
const data = await window.__productDataPromise; // already in flight or resolved!
Expected savings: up to 700ms (API call starts during JS parse/execution instead of after)

FIX 2: Parallelize independent requests
// ❌ BEFORE: sequential
const product  = await fetchProduct(id);
const reviews  = await fetchReviews(id);       // waits for product first — unnecessary
const related  = await fetchRelated(id);        // waits for reviews first — unnecessary

// ✅ AFTER: parallel (none depend on each other)
const [product, reviews, related] = await Promise.all([
  fetchProduct(id),
  fetchReviews(id),
  fetchRelated(id),
]);
Expected savings: ~1600ms (reviews + related now run concurrently
with product fetch instead of after it)

FIX 3 (more advanced): Streaming SSR or partial hydration
  If using a framework with streaming SSR (Next.js App Router, Remix):
  Render the product info as soon as ITS data resolves, stream reviews
  and related products as separate Suspense boundaries that resolve
  independently and "pop in" as ready — perceived performance improves
  even more than total time, because users see SOMETHING immediately.

NEW TIMELINE AFTER FIXES:
  index.html:              0-200ms
  main.js + product fetch: 200-900ms (fetch starts in parallel with JS load via Fix 1)
  product/reviews/related: 900-1900ms (parallel via Fix 2 — bound by SLOWEST one)
  Final render:            1900-2400ms

  TOTAL: ~2.4s (down from 4s) — a 40% improvement
  With streaming SSR: perceived content appears even faster, as low as ~1s
  for the product info specifically.
```

</details>

---

## Layout and Paint Performance

### Exercise 5.1 — Fix Layout Thrashing

```javascript
// This code measures and repositions 100 elements, causing severe
// layout thrashing (forced synchronous layout repeated 100 times).
// Fix it.

function repositionElements(elements) {
  elements.forEach((el) => {
    const height = el.offsetHeight; // READ
    el.style.top = `${height * 2}px`; // WRITE
  });
}
```

<details>
<summary>Solution</summary>

```javascript
// PROBLEM: interleaved read/write forces layout recalculation on EVERY
// iteration. Reading offsetHeight after a previous iteration's write
// invalidates the cached layout, forcing the browser to recompute layout
// synchronously before it can return the height — 100 forced layouts.

// FIX: batch all reads first, then all writes (read-write separation)
function repositionElements(elements) {
  // PHASE 1: read everything first (one layout calculation, if any)
  const heights = elements.map((el) => el.offsetHeight);

  // PHASE 2: write everything (no interleaved reads — no forced layout)
  elements.forEach((el, i) => {
    el.style.top = `${heights[i] * 2}px`;
  });
}

// Result: instead of 100 forced synchronous layouts, this triggers AT MOST
// 1 layout calculation (when the first offsetHeight is read) — the browser
// can serve subsequent reads from the same cached layout since nothing
// was written in between.

// For React: this read-write separation is why libraries like Framer Motion
// and React's own DOM operations batch reads and writes internally.

// If you need a measure-then-mutate pattern repeatedly (e.g., in an
// animation loop), consider:
function useMeasureBeforeWrite(elementsRef) {
  useLayoutEffect(() => {
    // useLayoutEffect runs after DOM mutations but before paint —
    // ideal for read-then-write patterns that must happen before the
    // user sees the previous frame
    const heights = elementsRef.current.map((el) => el.offsetHeight); // batch read
    elementsRef.current.forEach((el, i) => {
      // batch write
      el.style.top = `${heights[i] * 2}px`;
    });
  });
}
```

</details>

---

## Quick Diagnostic Scenarios

```
SCENARIO A: "Scrolling is janky on a page with a sticky header that
            changes opacity based on scroll position."
DIAGNOSIS: Likely reading scrollY and writing opacity in the same scroll
           handler without batching, OR using a non-passive scroll listener
           that blocks scrolling, OR animating a paint-triggering property.
FIX: requestAnimationFrame to batch the opacity update; ensure listener
     is passive: addEventListener('scroll', handler, { passive: true });
     animate opacity only (compositor-friendly).

SCENARIO B: "Initial page load score is good in Lighthouse but users
            report the page feels slow when they actually interact with it."
DIAGNOSIS: Good FCP/LCP doesn't mean low main-thread blocking. Check
           Total Blocking Time (TBT) and Time to Interactive (TTI).
           Likely cause: large JS execution after paint, hydration cost.
FIX: code split, defer non-critical JS, reduce hydration cost (partial
     hydration / islands architecture if using SSR).

SCENARIO C: "A data table with 50 columns and 200 rows causes the browser
            tab to use 800MB of RAM and eventually crash."
DIAGNOSIS: 10,000 DOM nodes (50 × 200) plus likely non-virtualized
           rendering, possible memory leak from event listeners per cell,
           or accumulating state without cleanup.
FIX: virtualize BOTH rows and columns; profile with Memory panel heap
     snapshots to rule out a leak; ensure cell components don't each
     attach their own document-level listeners.
```

---

## 🔗 Related Topics

- [`performance/`](../performance/) — Full performance section
- [`anti-patterns/03-premature-optimization.md`](../anti-patterns/03-premature-optimization.md)
- [`debugging/01-chrome-devtools.md`](../debugging/01-chrome-devtools.md)
- [`debugging/02-react-devtools.md`](../debugging/02-react-devtools.md)

---

<div align="center">

**`exercises/` section complete!** 🎉

All 3 exercises files done:
`01-javascript-exercises.md` · `02-react-exercises.md` · **`03-performance-exercises.md`** ✓

**Next section:** [`challenges/`](../challenges/) →

</div>
