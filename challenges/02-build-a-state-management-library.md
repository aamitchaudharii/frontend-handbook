# 02 — Challenge: Build a State Management Library from Scratch

> **"Building your own Redux or Zustand isn't about replacing them — it's about understanding exactly what problem they solve and exactly what they cost to solve it. Once you've implemented the subscription model, the selector optimization, and the middleware chain yourself, you'll read their source code as a reader who already knows the answer, not a tourist."**

This challenge builds a small but genuinely usable state management library in stages, mirroring the real design decisions behind Redux, Zustand, and Jotai. By the end you'll understand the subscription model, why selectors matter for performance, how middleware composition works, and the tradeoffs between centralized and atomic state models.

---

## The Goal

Build `createStore()` — a minimal state container with subscriptions, selector-based reads, middleware support, and a React binding hook — that a real application could use for non-trivial state management.

---

## Stage 1 — The Minimal Store (Pub/Sub Foundation)

### Requirements

- `createStore(initialState)` returns a store object
- `store.getState()` returns the current state
- `store.setState(partialOrFn)` updates state (merging objects, like React's setState)
- `store.subscribe(listener)` registers a listener called on every state change, returns an unsubscribe function

<details>
<summary>Hints</summary>

- This is fundamentally the Observer pattern — a Set of listener functions
- setState should support both a partial object AND a function `(prevState) => partialState`
- Don't forget: what happens if a listener subscribes/unsubscribes WHILE you're notifying listeners?
</details>

<details>
<summary>Solution</summary>

```javascript
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();

  function getState() {
    return state;
  }

  function setState(partial) {
    const partialState =
      typeof partial === "function" ? partial(state) : partial;
    state = { ...state, ...partialState }; // shallow merge, like React's setState
    listeners.forEach((listener) => listener(state));
  }

  function subscribe(listener) {
    listeners.add(listener);
    return () => listeners.delete(listener); // unsubscribe function
  }

  return { getState, setState, subscribe };
}

// Usage:
const store = createStore({ count: 0, user: null });
const unsubscribe = store.subscribe((state) =>
  console.log("State changed:", state),
);
store.setState({ count: 1 }); // logs: { count: 1, user: null }
store.setState((s) => ({ count: s.count + 1 })); // logs: { count: 2, user: null }
unsubscribe();
```

**Design note on the "iterating while modifying" problem:** Using a `Set` for listeners, and iterating with `forEach`, is safe for additions during iteration (new additions won't be visited in the current `forEach` pass per the spec) but removal mid-iteration of an entry not yet visited correctly skips it. This mirrors React's own internal listener handling — using `Set` rather than an array avoids duplicate-registration bugs and gives O(1) removal.

</details>

---

## Stage 2 — Selectors and Preventing Unnecessary Re-renders

### The Problem This Introduces

Stage 1's `subscribe` calls every listener on **every** state change — even if the listener only cares about one field that didn't change. In a React binding, this means every component re-renders on every store update, regardless of what slice of state they actually use.

### Requirement

Add selector support: a hook `useStore(store, selector)` that only triggers a re-render when the **selected slice** of state actually changes — not on every store update.

<details>
<summary>Hints</summary()>

- You need to compare the PREVIOUS selected value to the NEW selected value after each state change
- For object/array selections, "changed" likely means reference inequality (you'll document this limitation, just like React's own state comparisons)
- `useSyncExternalStore` (React 18+) is the correct primitive for subscribing to external stores correctly with concurrent rendering — don't roll your own `useEffect` + `useState` subscription, it has tearing bugs under concurrent features
</details>

<details>
<summary>Solution</summary>

```javascript
import { useSyncExternalStore, useCallback, useRef } from "react";

function useStore(store, selector = (s) => s) {
  // useSyncExternalStore: the React-blessed way to subscribe to external
  // stores. It handles concurrent rendering correctly (no tearing) and
  // lets us implement our own subscribe/getSnapshot logic.

  const getSnapshot = useCallback(
    () => selector(store.getState()),
    [store, selector],
  );

  return useSyncExternalStore(
    store.subscribe, // subscribe function: (callback) => unsubscribe
    getSnapshot, // getSnapshot: returns current selected value
  );
}

// Usage:
const store = createStore({ count: 0, user: { name: "Alice" } });

function Counter() {
  // Only re-renders when `count` changes, NOT when `user` changes
  const count = useStore(store, (state) => state.count);
  return (
    <button onClick={() => store.setState((s) => ({ count: s.count + 1 }))}>
      {count}
    </button>
  );
}

function UserProfile() {
  // Only re-renders when `user` reference changes, NOT when `count` changes
  const user = useStore(store, (state) => state.user);
  return <div>{user.name}</div>;
}
```

**Why `useSyncExternalStore` instead of manual `useEffect` + `useState`?** A naive implementation —

```javascript
// ❌ DON'T do this — has a tearing bug
function useStoreNaive(store, selector) {
  const [value, setValue] = useState(() => selector(store.getState()));
  useEffect(() => {
    return store.subscribe(() => setValue(selector(store.getState())));
  }, [store, selector]);
  return value;
}
```

— has a subtle bug under React 18's concurrent rendering: if the store updates DURING a concurrent render (between when React starts rendering and when it commits), this component can read a STALE value because the `useEffect` subscription hasn't run yet (effects run after commit). `useSyncExternalStore` is specifically designed to solve this "tearing" problem by forcing a synchronous re-check of the snapshot before committing.

**Critical bug to watch for:** The `getSnapshot` function MUST return a value that's referentially stable when the underlying data hasn't changed, or `useSyncExternalStore` will think the value changed every time and infinite-loop. If `selector` returns a NEW object/array on every call (e.g., `(state) => ({ count: state.count })` creates a new object every time), this breaks. The fix: ensure selectors return raw values or stable references — or add memoization to the selector itself (Stage 3 addresses this more robustly).

</details>

---

## Stage 3 — Memoized Selectors (Avoiding the New-Object-Every-Call Trap)

### New Requirement

Support selectors that compute derived values (not just direct field access) without causing infinite re-renders or unnecessary work — e.g., `(state) => state.items.filter(i => i.active)`.

<details>
<summary>Hints</summary>

- You need a way to compare the OUTPUT of a selector across calls, not just rely on reference equality if the selector creates new objects
- This is essentially what `reselect` (the Redux selector memoization library) does
- Consider: cache the last selector inputs and outputs; if inputs are the same (by reference), return the cached output instead of recomputing
</details>

<details>
<summary>Solution</summary>

```javascript
// A simple memoized selector creator (simplified "reselect")
function createSelector(inputSelectors, resultFn) {
  let lastInputs = null;
  let lastResult = null;

  return function selector(state) {
    const inputs = inputSelectors.map((fn) => fn(state));

    const inputsChanged =
      !lastInputs || inputs.some((input, i) => input !== lastInputs[i]);

    if (inputsChanged) {
      lastResult = resultFn(...inputs);
      lastInputs = inputs;
    }

    return lastResult; // same reference if inputs haven't changed
  };
}

// Usage: derive a filtered, computed value without recomputing on every call
const selectActiveItems = createSelector(
  [(state) => state.items], // input selector: raw items array
  (items) => items.filter((i) => i.active), // result function: derive active items
);

function ActiveItemsList() {
  // selectActiveItems only recomputes when state.items reference changes
  // (e.g., when an item is added/removed) — NOT on every store update
  // (e.g., when an unrelated field like `count` changes)
  const activeItems = useStore(store, selectActiveItems);
  return activeItems.map((item) => <Item key={item.id} item={item} />);
}

// Without this memoization: selectActiveItems would create a NEW filtered
// array on EVERY call (even when state.items hasn't changed), and since
// useSyncExternalStore compares by reference, this would cause infinite
// re-renders — exactly the trap mentioned in Stage 2.
```

**Design insight:** This memoization pattern — cache based on shallow-compared INPUTS, not the selector's output — is the core idea behind `reselect` and is also conceptually similar to how `useMemo` works (compare deps, reuse cached value if unchanged). The key difference from `useMemo` is that this memoization lives OUTSIDE React entirely, in the selector itself, so the SAME memoized selector instance can be shared and reused efficiently across multiple components that need the same derived data.

</details>

---

## Stage 4 — Middleware (Composable Side Effects on Every Update)

### New Requirement

Add middleware support: functions that can intercept, log, modify, or short-circuit `setState` calls — supporting use cases like logging, async actions, persistence to localStorage, and devtools integration.

<details>
<summary>Hints</summary>

- This is the same "onion" composition pattern used in Express middleware and Redux middleware
- Each middleware wraps the NEXT middleware (or the final setState), receiving (store) => (next) => (action) => result
- Think about what each middleware needs access to: the store's getState/setState, and a reference to "call the next thing in the chain"
</details>

<details>
<summary>Solution</summary>

```javascript
function createStore(initialState, middlewares = []) {
  let state = initialState;
  const listeners = new Set();

  function getState() {
    return state;
  }

  // The "raw" setState, before middleware wrapping
  function rawSetState(partial) {
    const partialState =
      typeof partial === "function" ? partial(state) : partial;
    state = { ...state, ...partialState };
    listeners.forEach((listener) => listener(state));
  }

  function subscribe(listener) {
    listeners.add(listener);
    return () => listeners.delete(listener);
  }

  const storeApi = { getState, subscribe };

  // Compose middleware around rawSetState, onion-style:
  // middlewares = [logger, persistence]
  // Final chain: logger(persistence(rawSetState))
  // Call order on setState: logger runs first, calls next (persistence),
  // which calls next (rawSetState)
  const setState = middlewares.reduceRight(
    (next, middleware) => middleware(storeApi)(next),
    rawSetState,
  );

  storeApi.setState = setState;
  return storeApi;
}

// Example middleware: logger
const logger = (store) => (next) => (partial) => {
  const prevState = store.getState();
  next(partial); // call the next middleware (or rawSetState)
  console.log("Previous:", prevState, "Next:", store.getState());
};

// Example middleware: localStorage persistence
const persist = (key) => (store) => (next) => (partial) => {
  next(partial);
  localStorage.setItem(key, JSON.stringify(store.getState()));
};

// Example middleware: action-based logging (named actions instead of raw partials)
function withDevtools(store) {
  return (next) => (partial) => {
    const label =
      typeof partial === "function" ? partial.name || "anonymous" : "setState";
    console.group(label);
    next(partial);
    console.groupEnd();
  };
}

// Usage: compose multiple middlewares
const store = createStore({ count: 0 }, [
  logger,
  persist("app-state"),
  withDevtools,
]);

store.setState({ count: 1 });
// Execution order: logger wraps persist wraps withDevtools wraps rawSetState
// logger logs prev/next state
// persist saves the FINAL state to localStorage after the update completes
// withDevtools groups the console output
```

**Why `reduceRight`?** Middleware composition is fundamentally function composition, and the order matters: the FIRST middleware in your array should be the OUTERMOST wrapper (it runs first and last — wrapping everything else). `reduceRight` processes the array from right to left, so the LAST middleware gets wrapped first (becomes innermost), and the FIRST middleware ends up wrapping everything (becomes outermost). This exactly matches how Redux's `applyMiddleware` works, and is the same pattern you'd use to compose Express/Koa middleware.

</details>

---

## Stage 5 — Async Actions and Optimistic Updates

### New Requirement

Support async state transitions — e.g., "set loading, fetch data, then either set success or error state" — as a first-class pattern, with optimistic update + rollback support.

<details>
<summary>Hints</summary>

- This doesn't require new core store features — it's a PATTERN built on what you already have
- For optimistic updates: apply the change immediately, keep a snapshot of the previous state, and roll back to that snapshot if the async operation fails
</details>

<details>
<summary>Solution</summary>

```javascript
// Async action pattern (no new store primitives needed)
function createAsyncAction(store, asyncFn) {
  return async function (...args) {
    store.setState({ status: "loading", error: null });
    try {
      const result = await asyncFn(...args);
      store.setState({ status: "success", data: result });
      return result;
    } catch (error) {
      store.setState({ status: "error", error: error.message });
      throw error;
    }
  };
}

// Usage:
const store = createStore({ status: "idle", data: null, error: null });
const fetchUser = createAsyncAction(store, (id) => api.getUser(id));
await fetchUser("user-42");

// Optimistic update with rollback
function createOptimisticAction(store, optimisticUpdate, asyncFn) {
  return async function (...args) {
    const snapshot = store.getState(); // save for rollback

    store.setState(optimisticUpdate(...args)); // apply immediately

    try {
      const result = await asyncFn(...args);
      return result;
    } catch (error) {
      store.setState(snapshot); // rollback to the pre-optimistic state
      throw error;
    }
  };
}

// Usage: optimistic "like" button
const toggleLike = createOptimisticAction(
  store,
  (postId) => (state) => ({
    posts: state.posts.map((p) =>
      p.id === postId
        ? { ...p, liked: !p.liked, likeCount: p.likeCount + (p.liked ? -1 : 1) }
        : p,
    ),
  }),
  (postId) => api.toggleLike(postId), // actual server call
);

await toggleLike("post-1"); // UI updates instantly; rolls back if server call fails
```

**Design insight:** Notice that NONE of this required adding async-specific machinery to the store itself — `setState` is synchronous and simple, and async coordination happens entirely in the calling code using ordinary `async/await`. This is a deliberate design choice that mirrors Zustand's philosophy (in contrast to Redux's historical reliance on middleware like `redux-thunk` or `redux-saga` for async — though modern Redux Toolkit has simplified this significantly with `createAsyncThunk`).

</details>

---

## Stage 6 — Stretch Goals

```
1. COMPUTED/DERIVED STATE THAT AUTO-UPDATES (Jotai-style atoms)
   Instead of a single big store, implement small independent "atoms"
   that can depend on each other, recomputing automatically when their
   dependencies change — without a central store object at all.

2. TIME-TRAVEL DEBUGGING
   Keep a history of all states (or all setState calls). Implement
   `store.undo()` and `store.redo()` that step through history.
   Consider: how do you handle this efficiently for large state objects
   without keeping full copies of everything? (Structural sharing /
   Immer-style patches are the production answer.)

3. CROSS-TAB SYNC
   Use the BroadcastChannel API or the `storage` event to synchronize
   store state across multiple browser tabs.

4. SLICE PATTERN (Redux Toolkit style)
   Implement a `createSlice({ name, initialState, reducers })` helper
   that generates action creators and a reducer automatically from a
   simple object of "case reducer" functions — without requiring
   manual action type strings.

5. PERSISTENCE WITH MIGRATION
   Build a persistence middleware that can handle SCHEMA CHANGES —
   if a user has old state shape saved in localStorage from a previous
   app version, migrate it to the new shape on load rather than crashing
   or silently ignoring the saved data.
```

---

## What You Should Have Learned

```
1. State management libraries are fundamentally the Observer pattern with
   careful attention to WHEN listeners are notified and HOW consumers
   avoid over-subscribing to changes they don't care about.

2. Selector-based reads are the mechanism for fine-grained reactivity:
   instead of "re-render on any change," you get "re-render only when
   YOUR slice changes" — this is the difference between Context's
   coarse-grained updates and a proper state library's fine-grained ones.

3. useSyncExternalStore exists specifically to make external store
   subscriptions safe under React's concurrent rendering — rolling your
   own with useEffect + useState has a real tearing bug, not a theoretical one.

4. Middleware composition is function composition with a specific call
   order convention (onion model) — understanding this pattern transfers
   directly to Express, Koa, and any other "chain of interceptors" system.

5. Async state management doesn't need special primitives in the store
   itself — async/await plus a synchronous setState is sufficient, with
   optimistic updates handled by snapshotting state before the optimistic
   change and rolling back on failure.
```

---

## 🔗 Related Topics

- [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md)
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md)
- [`javascript-core/12-design-patterns.md`](../javascript-core/12-design-patterns.md)

---

<div align="center">

**Next:** [`challenges/03-build-a-mini-react.md`](./03-build-a-mini-react.md) →

</div>
