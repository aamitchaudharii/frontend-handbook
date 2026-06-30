# 02 — React Exercises

> **"React exercises that matter aren't about memorizing API syntax — they're about internalizing the mental model: state drives UI, effects synchronize with the outside world, and performance comes from understanding what triggers what. Build these from scratch and the patterns become intuition."**

These exercises cover hooks, component patterns, performance optimization, and common interview-style component-building tasks. Each includes the problem, hints toward the right mental model, and a complete reference solution.

---

## Custom Hooks

### Exercise 1.1 — Implement `useToggle`

```jsx
// Implement a useToggle hook:
// - Returns [value, toggle, setValue]
// - toggle() flips the boolean
// - setValue(bool) sets it explicitly

function useToggle(initial = false) {
  // your implementation
}

// Test:
function Component() {
  const [isOpen, toggle, setOpen] = useToggle(false);
  return (
    <>
      <button onClick={toggle}>{isOpen ? "Close" : "Open"}</button>
      <button onClick={() => setOpen(true)}>Force Open</button>
    </>
  );
}
```

<details>
<summary>Solution</summary>

```jsx
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle, setValue];
}

// Concept: useCallback with functional updater means toggle never needs
// to be recreated — it doesn't capture `value` in its closure at all.
```

</details>

---

### Exercise 1.2 — Implement `usePrevious`

```jsx
// Implement usePrevious(value) that returns the value from the PREVIOUS render.
// Returns undefined on the first render.

function usePrevious(value) {
  // your implementation
}

// Test:
function Counter() {
  const [count, setCount] = useState(0);
  const previousCount = usePrevious(count);
  return (
    <div>
      <p>
        Now: {count}, Before: {previousCount}
      </p>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
    </div>
  );
}
```

<details>
<summary>Solution</summary>

```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value; // runs AFTER render, storing this render's value
  });
  return ref.current; // during THIS render, holds value from LAST render
}

// Concept: timing matters. The effect runs after the DOM commit, updating
// ref.current to the CURRENT value. But during render (before the effect runs),
// ref.current still has the PREVIOUS render's value — that's what gets returned.
```

</details>

---

### Exercise 1.3 — Implement `useDebounce`

```jsx
// Implement useDebounce(value, delay) that returns a debounced version
// of `value` — it only updates after `delay` ms have passed without changes.

function useDebounce(value, delay) {
  // your implementation
}

// Test:
function SearchBox() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      console.log("Searching for:", debouncedQuery); // fires 300ms after typing stops
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

<details>
<summary>Solution</summary>

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer); // cancel if value changes before delay elapses
  }, [value, delay]);

  return debouncedValue;
}

// Concept: cleanup function cancels the PREVIOUS timer whenever value changes,
// effectively resetting the debounce timer on every keystroke.
// Only when `value` stops changing for `delay` ms does setDebouncedValue actually fire.
```

</details>

---

### Exercise 1.4 — Implement `useLocalStorage`

```jsx
// Implement useLocalStorage(key, initialValue) — works like useState,
// but persists the value to localStorage and reads it back on mount.

function useLocalStorage(key, initialValue) {
  // your implementation
}

// Test:
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage("theme", "light");
  return (
    <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
      Current: {theme}
    </button>
  );
}
// Refreshing the page should preserve the last selected theme
```

<details>
<summary>Solution</summary>

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored !== null ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback(
    (newValue) => {
      setValue((prev) => {
        const resolved =
          newValue instanceof Function ? newValue(prev) : newValue;
        try {
          localStorage.setItem(key, JSON.stringify(resolved));
        } catch (err) {
          console.error(`Failed to save ${key} to localStorage`, err);
        }
        return resolved;
      });
    },
    [key],
  );

  return [value, setStoredValue];
}

// Concepts:
// Lazy initial state (function passed to useState) avoids reading localStorage on every render
// Functional updater support (newValue instanceof Function) mirrors useState's API
// try/catch handles private browsing mode where localStorage can throw
```

</details>

---

## Component Building

### Exercise 2.1 — Build a Controlled Modal

```jsx
// Build a Modal component that:
// - Renders via a portal to document.body
// - Closes on Escape key or backdrop click
// - Traps focus within the modal while open
// - Restores focus to the trigger element on close
// - Prevents body scroll while open

function Modal({ isOpen, onClose, children }) {
  // your implementation
}
```

<details>
<summary>Solution</summary>

```jsx
import { createPortal } from "react-dom";

function Modal({ isOpen, onClose, children }) {
  const modalRef = useRef(null);
  const previousFocusRef = useRef(null);

  // Save the trigger element's focus, restore on close
  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement;
      modalRef.current?.focus();
    } else {
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  // Escape key closes the modal
  useEffect(() => {
    if (!isOpen) return;
    function handleKeyDown(e) {
      if (e.key === "Escape") onClose();
    }
    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [isOpen, onClose]);

  // Prevent body scroll while open
  useEffect(() => {
    if (!isOpen) return;
    const originalOverflow = document.body.style.overflow;
    document.body.style.overflow = "hidden";
    return () => {
      document.body.style.overflow = originalOverflow;
    };
  }, [isOpen]);

  // Focus trap: keep Tab navigation within the modal
  function handleKeyDown(e) {
    if (e.key !== "Tab") return;
    const focusable = modalRef.current.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])',
    );
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();
    }
  }

  if (!isOpen) return null;

  return createPortal(
    <div
      className="modal-backdrop"
      onClick={(e) => {
        if (e.target === e.currentTarget) onClose();
      }}
    >
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        tabIndex={-1}
        className="modal-content"
        onKeyDown={handleKeyDown}
      >
        {children}
      </div>
    </div>,
    document.body,
  );
}

// Concepts tested:
// Portals (rendering outside the parent DOM hierarchy)
// Focus management (accessibility — trap and restore)
// Effect cleanup (always restore body overflow, remove listeners)
// Event delegation (backdrop click vs content click distinction)
```

</details>

---

### Exercise 2.2 — Build an Infinite Scroll List

```jsx
// Build an InfiniteScrollList that:
// - Fetches and renders pages of items
// - Loads the next page when the user scrolls near the bottom
// - Shows a loading indicator while fetching
// - Stops fetching when there are no more pages

function InfiniteScrollList({ fetchPage }) {
  // fetchPage(pageNumber) returns Promise<{ items: T[], hasMore: boolean }>
  // your implementation
}
```

<details>
<summary>Solution</summary>

```jsx
function InfiniteScrollList({ fetchPage }) {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(0);
  const [isLoading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const sentinelRef = useRef(null);

  // Load a page
  const loadPage = useCallback(
    async (pageNumber) => {
      if (isLoading || !hasMore) return;
      setLoading(true);
      try {
        const { items: newItems, hasMore: more } = await fetchPage(pageNumber);
        setItems((prev) => [...prev, ...newItems]);
        setHasMore(more);
      } finally {
        setLoading(false);
      }
    },
    [fetchPage, isLoading, hasMore],
  );

  // Initial load
  useEffect(() => {
    loadPage(0);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []); // intentionally run once on mount

  // IntersectionObserver: detect when sentinel enters viewport
  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasMore && !isLoading) {
          setPage((p) => {
            const next = p + 1;
            loadPage(next);
            return next;
          });
        }
      },
      { rootMargin: "200px" }, // trigger 200px before reaching the sentinel
    );

    observer.observe(sentinel);
    return () => observer.disconnect();
  }, [hasMore, isLoading, loadPage]);

  return (
    <div>
      {items.map((item) => (
        <ItemRow key={item.id} item={item} />
      ))}
      <div ref={sentinelRef} style={{ height: 1 }} />
      {isLoading && <Spinner />}
      {!hasMore && <p>No more items.</p>}
    </div>
  );
}

// Concepts tested:
// IntersectionObserver for scroll-triggered loading (better than scroll event listeners)
// Avoiding duplicate fetches (isLoading guard)
// rootMargin to pre-fetch before the user actually reaches the bottom
// Proper cleanup of the observer
```

</details>

---

## Performance Optimization

### Exercise 3.1 — Fix the Re-render Problem

```jsx
// This component re-renders ProductCard for EVERY product whenever ANY
// product's "favorite" status changes. Fix it so only the changed
// ProductCard re-renders.

function ProductList({ products }) {
  const [favorites, setFavorites] = useState(new Set());

  function toggleFavorite(id) {
    setFavorites(prev => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  }

  return (
    <div>
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          isFavorite={favorites.has(product.id)}
          onToggleFavorite={() => toggleFavorite(product.id)}
        />
      ))}
    </div>
  );
}

function ProductCard({ product, isFavorite, onToggleFavorite }) {
  console.log(`Rendering ${product.name}`); // logs for ALL products on ANY toggle
  return (/* render */);
}
```

<details>
<summary>Solution</summary>

```jsx
// Problem: `onToggleFavorite={() => toggleFavorite(product.id)}` creates a
// NEW function on every ProductList render. Even with React.memo on
// ProductCard, the new function reference breaks memoization for ALL cards.

// FIX 1: React.memo + stable handler via useCallback with the id passed at call time
const ProductCard = React.memo(function ProductCard({
  product,
  isFavorite,
  onToggleFavorite,
}) {
  console.log(`Rendering ${product.name}`);
  return (
    <div>
      <span>{product.name}</span>
      <button onClick={() => onToggleFavorite(product.id)}>
        {isFavorite ? "★" : "☆"}
      </button>
    </div>
  );
});

function ProductList({ products }) {
  const [favorites, setFavorites] = useState(new Set());

  // Stable function reference: doesn't change between renders
  const toggleFavorite = useCallback((id) => {
    setFavorites((prev) => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  }, []); // empty deps: uses functional updater, no closure over favorites needed

  return (
    <div>
      {products.map((product) => (
        <ProductCard
          key={product.id}
          product={product}
          isFavorite={favorites.has(product.id)}
          onToggleFavorite={toggleFavorite} // stable reference, ID passed at call time
        />
      ))}
    </div>
  );
}

// Now: toggling product A's favorite status only changes `favorites` (new Set)
// Each ProductCard's `isFavorite` prop is recalculated, but only the CHANGED
// product's isFavorite value actually differs → only that one re-renders
// (assuming product and onToggleFavorite props are stable, which they are)

// Verify with React DevTools Profiler: "Why did this render?" should show
// "Props changed: isFavorite" ONLY for the toggled product, others show no render
```

</details>

---

### Exercise 3.2 — Virtualize a Large List

```jsx
// Implement a SIMPLE virtualized list from scratch (without a library).
// Given 10,000 items, only render the ones visible in the viewport (+ overscan).

function VirtualList({ items, itemHeight, containerHeight, renderItem }) {
  // your implementation
}
```

<details>
<summary>Solution</summary>

```jsx
function VirtualList({
  items,
  itemHeight,
  containerHeight,
  renderItem,
  overscan = 3,
}) {
  const [scrollTop, setScrollTop] = useState(0);
  const containerRef = useRef(null);

  const totalHeight = items.length * itemHeight;

  // Determine which items are visible
  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const visibleCount = Math.ceil(containerHeight / itemHeight) + overscan * 2;
  const endIndex = Math.min(items.length, startIndex + visibleCount);

  const visibleItems = items.slice(startIndex, endIndex);

  function handleScroll(e) {
    setScrollTop(e.target.scrollTop);
  }

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      style={{
        height: containerHeight,
        overflowY: "auto",
        position: "relative",
      }}
    >
      {/* Spacer div: gives the scrollable area the correct total height */}
      <div style={{ height: totalHeight, position: "relative" }}>
        {visibleItems.map((item, i) => {
          const actualIndex = startIndex + i;
          return (
            <div
              key={actualIndex}
              style={{
                position: "absolute",
                top: actualIndex * itemHeight,
                height: itemHeight,
                width: "100%",
              }}
            >
              {renderItem(item, actualIndex)}
            </div>
          );
        })}
      </div>
    </div>
  );
}

// Usage:
<VirtualList
  items={tenThousandItems}
  itemHeight={50}
  containerHeight={500}
  renderItem={(item) => <div>{item.name}</div>}
/>;

// Concepts:
// Only DOM nodes within the visible range (+overscan) are rendered
// Absolute positioning places each item at its correct scroll offset
// Spacer div gives the scrollbar the correct proportions for 10,000 items
// overscan renders a few extra items above/below viewport to avoid blank flashes during fast scroll
```

</details>

---

## State Management Patterns

### Exercise 4.1 — Build a useReducer-based Form

```jsx
// Build a form using useReducer that handles:
// - Field value updates
// - Field-level validation errors
// - Form-level submission state (idle, submitting, success, error)

// Define the reducer and action types yourself.
```

<details>
<summary>Solution</summary>

```jsx
const initialState = {
  values: { email: "", password: "" },
  errors: {},
  status: "idle", // idle | submitting | success | error
};

function formReducer(state, action) {
  switch (action.type) {
    case "FIELD_CHANGE":
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: undefined }, // clear error on change
      };

    case "SET_ERRORS":
      return { ...state, errors: action.errors };

    case "SUBMIT_START":
      return { ...state, status: "submitting", errors: {} };

    case "SUBMIT_SUCCESS":
      return { ...state, status: "success" };

    case "SUBMIT_ERROR":
      return { ...state, status: "error", errors: action.errors ?? {} };

    case "RESET":
      return initialState;

    default:
      return state;
  }
}

function validate(values) {
  const errors = {};
  if (!values.email.includes("@")) errors.email = "Invalid email";
  if (values.password.length < 8) errors.password = "Must be 8+ characters";
  return errors;
}

function LoginForm() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  function handleChange(field) {
    return (e) =>
      dispatch({ type: "FIELD_CHANGE", field, value: e.target.value });
  }

  async function handleSubmit(e) {
    e.preventDefault();
    const errors = validate(state.values);

    if (Object.keys(errors).length > 0) {
      dispatch({ type: "SET_ERRORS", errors });
      return;
    }

    dispatch({ type: "SUBMIT_START" });
    try {
      await authApi.login(state.values);
      dispatch({ type: "SUBMIT_SUCCESS" });
    } catch (err) {
      dispatch({ type: "SUBMIT_ERROR", errors: { form: err.message } });
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={state.values.email}
        onChange={handleChange("email")}
        placeholder="Email"
      />
      {state.errors.email && (
        <span className="error">{state.errors.email}</span>
      )}

      <input
        type="password"
        value={state.values.password}
        onChange={handleChange("password")}
        placeholder="Password"
      />
      {state.errors.password && (
        <span className="error">{state.errors.password}</span>
      )}

      {state.errors.form && <p className="error">{state.errors.form}</p>}

      <button type="submit" disabled={state.status === "submitting"}>
        {state.status === "submitting" ? "Logging in..." : "Log In"}
      </button>
    </form>
  );
}

// Concepts: useReducer for related state transitions, action-based updates,
// pure reducer function (easily unit-testable without React), explicit state machine
```

</details>

---

## Quick Practice Problems

```jsx
// PROBLEM 1: useArray hook for common array operations
function useArray(initial = []) {
  const [array, setArray] = useState(initial);
  return {
    array,
    push: (item) => setArray((a) => [...a, item]),
    remove: (index) => setArray((a) => a.filter((_, i) => i !== index)),
    update: (index, item) =>
      setArray((a) => a.map((x, i) => (i === index ? item : x))),
    clear: () => setArray([]),
  };
}

// PROBLEM 2: useClickOutside hook
function useClickOutside(ref, callback) {
  useEffect(() => {
    function handleClick(e) {
      if (ref.current && !ref.current.contains(e.target)) callback();
    }
    document.addEventListener("mousedown", handleClick);
    return () => document.removeEventListener("mousedown", handleClick);
  }, [ref, callback]);
}

// PROBLEM 3: useMediaQuery hook
function useMediaQuery(query) {
  const [matches, setMatches] = useState(
    () => window.matchMedia(query).matches,
  );
  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e) => setMatches(e.matches);
    mql.addEventListener("change", handler);
    return () => mql.removeEventListener("change", handler);
  }, [query]);
  return matches;
}

// PROBLEM 4: Compound component - simple Tabs
function Tabs({ children, defaultIndex = 0 }) {
  const [activeIndex, setActiveIndex] = useState(defaultIndex);
  return React.Children.map(children, (child, index) =>
    React.cloneElement(child, {
      isActive: index === activeIndex,
      onClick: () => setActiveIndex(index),
    }),
  );
}
```

---

## 🔗 Related Topics

- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md)
- [`patterns/05-compound-components.md`](../patterns/05-compound-components.md)
- [`anti-patterns/03-premature-optimization.md`](../anti-patterns/03-premature-optimization.md)
- [`exercises/03-performance-exercises.md`](./03-performance-exercises.md)
