# 02 — Custom Hooks

> **"A custom hook is a function, not a component, but it follows component-like rules: it can hold state, run effects, and compose other hooks — yet it produces no UI. This is React's mechanism for extracting and reusing STATEFUL LOGIC, something that was genuinely difficult before hooks existed. Mastering custom hooks means understanding exactly what they share and what they don't."**

Custom hooks are the primary mechanism for extracting and reusing stateful logic between React components. Before hooks, sharing stateful logic required render props or higher-order components, both of which added wrapper components to the tree and made code harder to follow. Custom hooks solve this elegantly: they're just functions that call other hooks, composable like any function, with zero added component nesting. This document covers hook design principles, common hook patterns, the rules that govern correct hook composition, and the testing strategies that keep custom hooks reliable.

---

## 📚 Table of Contents

1. [What Custom Hooks Actually Share](#1-what-custom-hooks-actually-share)
2. [The Rules of Hooks (and Why They Exist)](#2-the-rules-of-hooks-and-why-they-exist)
3. [Designing a Good Custom Hook API](#3-designing-a-good-custom-hook-api)
4. [Common Hook Patterns](#4-common-hook-patterns)
5. [Data Fetching Hooks](#5-data-fetching-hooks)
6. [Hooks That Wrap Browser APIs](#6-hooks-that-wrap-browser-apis)
7. [Composing Hooks](#7-composing-hooks)
8. [Hooks with Cleanup](#8-hooks-with-cleanup)
9. [Generic and Reusable Hook Design](#9-generic-and-reusable-hook-design)
10. [Testing Custom Hooks](#10-testing-custom-hooks)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. What Custom Hooks Actually Share

This is the single most misunderstood aspect of custom hooks:

```jsx
// Custom hooks share LOGIC, not STATE
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount((c) => c + 1), []);
  const decrement = useCallback(() => setCount((c) => c - 1), []);
  return { count, increment, decrement };
}

function ComponentA() {
  const { count, increment } = useCounter(0); // independent state instance
  return <button onClick={increment}>A: {count}</button>;
}

function ComponentB() {
  const { count, increment } = useCounter(0); // ANOTHER independent state instance
  return <button onClick={increment}>B: {count}</button>;
}

// ComponentA and ComponentB each get their OWN count state
// Incrementing A's counter does NOT affect B's counter
// They share the IMPLEMENTATION of counter logic, not a shared counter value

// This is exactly like calling the same function twice — each call
// gets its own local variables, even though it's the "same" function
```

```
MENTAL MODEL:
  useState inside a custom hook behaves exactly as if you'd written
  that useState call directly in the component.

  The custom hook is a thin abstraction layer — React doesn't even know
  "useCounter" exists as a concept. It just sees a sequence of useState/
  useEffect/etc. calls, in order, regardless of which function they're
  textually written in.
```

---

## 2. The Rules of Hooks (and Why They Exist)

```jsx
// RULE 1: Only call hooks at the top level (not in loops, conditions, nested functions)
// RULE 2: Only call hooks from React functions (components or other custom hooks)

// WHY THESE RULES EXIST: React tracks hook state by CALL ORDER, not by name
function Component() {
  const [a, setA] = useState(1); // hook call #1
  const [b, setB] = useState(2); // hook call #2
  useEffect(() => {
    /* ... */
  }); // hook call #3

  // React's internal fiber stores: [stateA, stateB, effectC] as a linked list
  // On re-render: React expects the SAME sequence of hook calls, in the SAME order
}

// ❌ Conditional hook call breaks this — call order can change between renders
function BadComponent({ condition }) {
  const [a, setA] = useState(1);
  if (condition) {
    const [b, setB] = useState(2); // ❌ sometimes called, sometimes not!
  }
  useEffect(() => {
    /* ... */
  });
  // If `condition` changes between renders: the hook call sequence shifts
  // React can no longer correctly map state to the right hook call
  // → "Rendered more hooks than during the previous render" error, or worse: silent bugs
}

// ✅ Move the condition INSIDE the hook, not around the hook call
function GoodComponent({ condition }) {
  const [a, setA] = useState(1);
  const [b, setB] = useState(2); // ALWAYS called
  useEffect(() => {
    if (condition) {
      // conditional logic INSIDE the hook is fine
    }
  });
}
```

---

## 3. Designing a Good Custom Hook API

```typescript
// ✅ Good hook design: clear inputs, clear outputs, single responsibility
function useDebounce<T>(value: T, delayMs: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delayMs);
    return () => clearTimeout(timer);
  }, [value, delayMs]);

  return debouncedValue;
}

// ❌ Hook trying to do too much — hard to test, hard to reuse partially
function useEverythingAboutSearch(query: string) {
  const debouncedQuery = useDebounce(query, 300);
  const { data, isLoading } = useQuery(["search", debouncedQuery], () =>
    searchApi(debouncedQuery),
  );
  const [history, setHistory] = useState<string[]>([]);
  const [filters, setFilters] = useState({});
  const analytics = useAnalytics();

  useEffect(() => {
    analytics.track("search", { query: debouncedQuery });
    setHistory((h) => [...h, debouncedQuery]);
  }, [debouncedQuery]);

  return { data, isLoading, history, filters, setFilters };
  // Too many concerns bundled: debouncing + fetching + history + filters + analytics
  // Hard to use just ONE piece of this in another context
}

// ✅ Decomposed: each concern is its own hook, composed where needed
function useSearchQuery(query: string) {
  const debouncedQuery = useDebounce(query, 300);
  return useQuery(["search", debouncedQuery], () => searchApi(debouncedQuery));
}

function useSearchHistory(query: string) {
  const [history, setHistory] = useState<string[]>([]);
  useEffect(() => {
    if (query) setHistory((h) => [...h, query]);
  }, [query]);
  return history;
}

// Components compose only what they need:
function SearchBar() {
  const [query, setQuery] = useState("");
  const { data, isLoading } = useSearchQuery(query); // only fetching, no history needed
  // ...
}

function SearchPageWithHistory() {
  const [query, setQuery] = useState("");
  const { data } = useSearchQuery(query);
  const history = useSearchHistory(query); // composes both
  // ...
}
```

---

## 4. Common Hook Patterns

### State + Setter Hooks

```typescript
function useToggle(
  initial = false,
): [boolean, () => void, (value: boolean) => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle, setValue];
}

function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback(
    (newValue: T | ((prev: T) => T)) => {
      setValue((prev) => {
        const resolved =
          newValue instanceof Function ? newValue(prev) : newValue;
        try {
          localStorage.setItem(key, JSON.stringify(resolved));
        } catch (err) {
          console.error(`Failed to persist ${key} to localStorage`, err);
        }
        return resolved;
      });
    },
    [key],
  );

  return [value, setStoredValue] as const;
}
```

### Previous Value Tracking

```typescript
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => {
    ref.current = value;
  }); // no dependency array — runs after every render, storing the LAST render's value
  return ref.current; // during THIS render, ref.current still holds the PREVIOUS value
}

// Usage: detect direction of change
function PriceDisplay({ price }: { price: number }) {
  const previousPrice = usePrevious(price);
  const direction = previousPrice === undefined ? 'none'
    : price > previousPrice ? 'up'
    : price < previousPrice ? 'down' : 'none';

  return <span className={`price price--${direction}`}>{price}</span>;
}
```

### Boolean State with Named Actions

```typescript
function useDisclosure(initial = false) {
  const [isOpen, setIsOpen] = useState(initial);
  return {
    isOpen,
    open:   useCallback(() => setIsOpen(true), []),
    close:  useCallback(() => setIsOpen(false), []),
    toggle: useCallback(() => setIsOpen(o => !o), []),
  };
}

// Usage: more readable than raw boolean + setter
function Modal() {
  const { isOpen, open, close } = useDisclosure();
  return (
    <>
      <button onClick={open}>Open Modal</button>
      {isOpen && <Dialog onClose={close} />}
    </>
  );
}
```

---

## 5. Data Fetching Hooks

```typescript
// Basic data fetching hook with cancellation, loading, error states
function useFetch<T>(url: string | null) {
  const [state, setState] = useState<{
    data: T | null;
    error: Error | null;
    loading: boolean;
  }>({ data: null, error: null, loading: !!url });

  useEffect(() => {
    if (!url) return;

    const controller = new AbortController();
    setState((s) => ({ ...s, loading: true, error: null }));

    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((data) => setState({ data, error: null, loading: false }))
      .catch((err) => {
        if (err.name !== "AbortError") {
          setState({ data: null, error: err, loading: false });
        }
      });

    return () => controller.abort();
  }, [url]);

  return state;
}

// Note: in production, prefer TanStack Query over building this from scratch —
// it handles caching, deduplication, refetching, and many edge cases this doesn't.
// This pattern is valuable to understand the underlying mechanics, and for cases
// where a full data-fetching library is overkill.
```

### Polling Hook

```typescript
function usePolling<T>(
  fetchFn: () => Promise<T>,
  intervalMs: number,
  enabled = true,
) {
  const [data, setData] = useState<T | null>(null);
  const savedFetchFn = useRef(fetchFn);
  savedFetchFn.current = fetchFn; // always call the LATEST version

  useEffect(() => {
    if (!enabled) return;

    let cancelled = false;

    async function poll() {
      try {
        const result = await savedFetchFn.current();
        if (!cancelled) setData(result);
      } catch (err) {
        console.error("Polling error:", err);
      }
    }

    poll(); // immediate first call
    const id = setInterval(poll, intervalMs);

    return () => {
      cancelled = true;
      clearInterval(id);
    };
  }, [intervalMs, enabled]);

  return data;
}
```

---

## 6. Hooks That Wrap Browser APIs

```typescript
// Window size with resize listener
function useWindowSize() {
  const [size, setSize] = useState(() => ({
    width:  typeof window !== 'undefined' ? window.innerWidth  : 0,
    height: typeof window !== 'undefined' ? window.innerHeight : 0,
  }));

  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}

// Media query hook
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() =>
    typeof window !== 'undefined' ? window.matchMedia(query).matches : false
  );

  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener('change', handler);
    setMatches(mql.matches); // sync in case it changed between render and effect
    return () => mql.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// Usage:
function ResponsiveComponent() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  return isMobile ? <MobileNav /> : <DesktopNav />;
}

// IntersectionObserver hook
function useInView(options?: IntersectionObserverInit) {
  const ref       = useRef<HTMLElement>(null);
  const [inView, setInView] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => setInView(entry.isIntersecting),
      options
    );
    observer.observe(el);
    return () => observer.disconnect();
  }, [options]);

  return [ref, inView] as const;
}

// Usage:
function LazyImage({ src }: { src: string }) {
  const [ref, inView] = useInView({ rootMargin: '200px' });
  return <div ref={ref}>{inView ? <img src={src} /> : <Placeholder />}</div>;
}
```

---

## 7. Composing Hooks

```typescript
// Hooks compose like functions — build complex behavior from simple pieces
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  useEffect(() => {
    const goOnline = () => setIsOnline(true);
    const goOffline = () => setIsOnline(false);
    window.addEventListener("online", goOnline);
    window.addEventListener("offline", goOffline);
    return () => {
      window.removeEventListener("online", goOnline);
      window.removeEventListener("offline", goOffline);
    };
  }, []);
  return isOnline;
}

function useSyncQueue<T>() {
  const [queue, setQueue] = useState<T[]>([]);
  const enqueue = useCallback((item: T) => setQueue((q) => [...q, item]), []);
  const dequeue = useCallback(() => setQueue((q) => q.slice(1)), []);
  return { queue, enqueue, dequeue };
}

// Compose: offline-aware sync queue
function useOfflineSync<T>(syncFn: (item: T) => Promise<void>) {
  const isOnline = useOnlineStatus(); // composed hook 1
  const { queue, enqueue, dequeue } = useSyncQueue<T>(); // composed hook 2

  useEffect(() => {
    if (!isOnline || queue.length === 0) return;

    let cancelled = false;
    async function processQueue() {
      while (queue.length > 0 && !cancelled) {
        await syncFn(queue[0]);
        dequeue();
      }
    }
    processQueue();
    return () => {
      cancelled = true;
    };
  }, [isOnline, queue, syncFn, dequeue]);

  return { isOnline, pendingCount: queue.length, enqueue };
}

// Final consumer: one hook call gets all the composed behavior
function useTaskSync() {
  return useOfflineSync<Task>(async (task) => {
    await api.tasks.create(task);
  });
}
```

---

## 8. Hooks with Cleanup

```typescript
// Every effect with a subscription, timer, or listener needs cleanup
function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: Window | HTMLElement = window
) {
  const savedHandler = useRef(handler);
  savedHandler.current = handler; // always latest handler, no stale closures

  useEffect(() => {
    const eventListener = (event: Event) => savedHandler.current(event as WindowEventMap[K]);
    element.addEventListener(eventName, eventListener);
    return () => element.removeEventListener(eventName, eventListener); // cleanup
  }, [eventName, element]);
}

// Usage: handler can be an inline arrow function, no stale closure issues
function ComponentWithKeyboardShortcut() {
  const [count, setCount] = useState(0);

  useEventListener('keydown', (e) => {
    if (e.key === 'ArrowUp') setCount(c => c + 1); // always sees latest setCount, fine since setCount is stable
  });

  return <div>{count}</div>;
}
```

```typescript
// WebSocket hook with proper cleanup
function useWebSocket(url: string) {
  const [lastMessage, setLastMessage] = useState<MessageEvent | null>(null);
  const [readyState, setReadyState] = useState<number>(WebSocket.CONNECTING);
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => setReadyState(ws.readyState);
    ws.onclose = () => setReadyState(ws.readyState);
    ws.onmessage = (event) => setLastMessage(event);

    return () => {
      ws.close(); // ALWAYS close on cleanup — prevents leaked connections
      wsRef.current = null;
    };
  }, [url]);

  const sendMessage = useCallback((data: string) => {
    wsRef.current?.send(data);
  }, []);

  return { lastMessage, readyState, sendMessage };
}
```

---

## 9. Generic and Reusable Hook Design

```typescript
// Generic hook with TypeScript generics for reusability across types
function useAsync<T, E = Error>(asyncFn: () => Promise<T>, immediate = true) {
  const [status, setStatus] = useState<
    "idle" | "pending" | "success" | "error"
  >("idle");
  const [value, setValue] = useState<T | null>(null);
  const [error, setError] = useState<E | null>(null);

  const execute = useCallback(async () => {
    setStatus("pending");
    setValue(null);
    setError(null);
    try {
      const result = await asyncFn();
      setValue(result);
      setStatus("success");
      return result;
    } catch (err) {
      setError(err as E);
      setStatus("error");
      throw err;
    }
  }, [asyncFn]);

  useEffect(() => {
    if (immediate) execute();
  }, [execute, immediate]);

  return {
    execute,
    status,
    value,
    error,
    isIdle: status === "idle",
    isLoading: status === "pending",
    isSuccess: status === "success",
    isError: status === "error",
  };
}

// Reusable for ANY async operation, any return type:
const { value: user, isLoading } = useAsync(() => fetchUser(userId));
const { execute: submitForm, isLoading: isSubmitting } = useAsync(
  () => api.submitForm(formData),
  false, // don't run immediately — call execute() manually
);
```

---

## 10. Testing Custom Hooks

```typescript
import { renderHook, act, waitFor } from "@testing-library/react";

describe("useCounter", () => {
  test("increments count", () => {
    const { result } = renderHook(() => useCounter(0));

    expect(result.current.count).toBe(0);

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  test("respects initial value", () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });
});

describe("useDebounce", () => {
  beforeEach(() => jest.useFakeTimers());
  afterEach(() => jest.useRealTimers());

  test("debounces value updates", () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: "a", delay: 300 } },
    );

    expect(result.current).toBe("a");

    rerender({ value: "ab", delay: 300 });
    expect(result.current).toBe("a"); // not yet updated

    act(() => jest.advanceTimersByTime(300));
    expect(result.current).toBe("ab"); // now updated
  });
});

// Testing hooks with async operations
describe("useFetch", () => {
  test("loads data successfully", async () => {
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ id: 1, name: "Test" }),
    });

    const { result } = renderHook(() => useFetch("/api/test"));

    expect(result.current.loading).toBe(true);

    await waitFor(() => expect(result.current.loading).toBe(false));

    expect(result.current.data).toEqual({ id: 1, name: "Test" });
    expect(result.current.error).toBeNull();
  });
});
```

---

## 11. Good Practices

### ✅ Prefix custom hooks with "use"

```typescript
// ✅ React's ESLint plugin relies on this convention to enforce Rules of Hooks
function useFormValidation(schema) {
  /* ... */
}
```

### ✅ Return objects for 3+ values, arrays for 2 related values

```typescript
// ✅ Array destructuring for tuple-like pairs (mirrors useState convention)
const [value, setValue] = useToggle();

// ✅ Object for many values (named access, order-independent)
const { data, isLoading, error, refetch } = useFetch(url);
```

### ✅ Use refs to avoid stale closures in long-lived effects

```typescript
// ✅ Ref pattern keeps the effect's dependency array minimal
const savedCallback = useRef(callback);
savedCallback.current = callback;
useEffect(() => {
  const id = setInterval(() => savedCallback.current(), 1000);
  return () => clearInterval(id);
}, []); // no need to include `callback` — ref always has the latest
```

---

## 12. Bad Practices

### ❌ Hooks that do too many unrelated things

```typescript
// ❌ One hook handling fetching, formatting, validation, AND analytics
function useEverything(url) {
  /* 200 lines doing 5 different things */
}

// ✅ Decompose into focused, composable hooks (see Section 3)
```

### ❌ Calling hooks conditionally

```jsx
// ❌ Violates Rules of Hooks
function Component({ shouldFetch }) {
  if (shouldFetch) {
    const { data } = useFetch("/api/data"); // ❌ conditional hook call
  }
}

// ✅ Always call the hook; pass the condition INTO the hook
function Component({ shouldFetch }) {
  const { data } = useFetch(shouldFetch ? "/api/data" : null);
}
```

---

## 13. Common Mistakes

### Mistake 1 — Forgetting that hooks don't share state across instances

```jsx
// ❌ Expecting two components using the same hook to share state
function useSharedCounter() {
  const [count, setCount] = useState(0); // NOT shared — new instance per call
  return [count, setCount];
}
// If you need TRUE shared state: use Context, or a module-level store (Zustand)
```

### Mistake 2 — Missing cleanup causing memory leaks

```typescript
// ❌ No cleanup — subscription persists after unmount
function useSubscription(emitter) {
  const [value, setValue] = useState(null);
  useEffect(() => {
    emitter.on("change", setValue);
    // missing: return () => emitter.off('change', setValue);
  }, [emitter]);
  return value;
}
```

### Mistake 3 — Returning a new object/array every render without memoization

```typescript
// ❌ Returns a new object every call — breaks consumer's memoization
function useConfig() {
  return { theme: "dark", locale: "en" }; // new object reference every render!
}
// Any component using this in a dependency array re-runs effects every render

// ✅ Memoize if the value is used in dependency arrays downstream
function useConfig() {
  return useMemo(() => ({ theme: "dark", locale: "en" }), []);
}
```

---

## 14. Interview-Level Explanation

> **"What do custom hooks actually share between components? How do you design a good custom hook?"**

**Strong answer:**

> "Custom hooks share logic, not state. When two components call the same custom hook, each call gets its own independent state — exactly as if you'd written the `useState` calls directly in each component. The custom hook is just a function that happens to call other hooks internally; React doesn't track 'useCounter' as a concept, it tracks the underlying sequence of `useState`/`useEffect` calls per component instance. This is the single biggest misconception people have — they expect custom hooks to provide shared state like a singleton, when actually you need Context or an external store for that.
>
> Designing a good custom hook starts with single responsibility. A hook should do one thing — debouncing, fetching, tracking online status — rather than bundling several concerns. This keeps hooks composable: you can combine `useDebounce` and `useFetch` in a search component without pulling in unrelated logic, and you can test each piece independently. When a hook needs to do more, compose multiple focused hooks together rather than writing one large hook.
>
> For API design, I follow the convention `useState` itself establishes: return a tuple — array destructuring — for closely related pairs like `[value, setValue]`, and return an object when there are three or more values, since named destructuring doesn't depend on argument order and reads better at the call site for things like `{ data, isLoading, error, refetch }`.
>
> Cleanup discipline matters a lot for hooks wrapping subscriptions, timers, or event listeners — every `useEffect` that sets something up needs to tear it down in its cleanup function, or you leak subscriptions across re-renders and unmounts. A common pattern to avoid stale closures in long-lived effects — like a WebSocket connection or an interval — is storing the latest callback in a ref and reading from the ref inside the effect, which lets you keep the effect's dependency array minimal without referencing a stale version of a function.
>
> The Rules of Hooks — only call hooks at the top level, only from React functions — exist because React tracks hook state purely by call order, using something like a linked list per component fiber. If a hook is called conditionally, the sequence of calls can differ between renders, and React can no longer correctly map stored state back to the right hook call. The fix is always to move the conditional logic inside the hook rather than around the hook call itself — call the hook unconditionally, and let the hook's internal logic decide what to do based on its arguments."

---

## 15. Exercises

### Exercise 1 — Build a useFetchWithRetry hook

Build a custom hook that fetches data, retries on failure with exponential backoff (max 3 attempts), and exposes loading/error/data states plus a manual retry function.

<details>
<summary>Solution</summary>

```typescript
function useFetchWithRetry<T>(url: string | null, maxRetries = 3) {
  const [state, setState] = useState<{
    data: T | null;
    error: Error | null;
    loading: boolean;
    attempt: number;
  }>({ data: null, error: null, loading: !!url, attempt: 0 });

  const fetchWithRetry = useCallback(async (signal: AbortSignal) => {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        setState(s => ({ ...s, loading: true, attempt }));
        const response = await fetch(url!, { signal });
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const data = await response.json();
        setState({ data, error: null, loading: false, attempt });
        return;
      } catch (err) {
        if ((err as Error).name === 'AbortError') return;

        if (attempt === maxRetries) {
          setState({ data: null, error: err as Error, loading: false, attempt });
          return;
        }

        // Exponential backoff before retrying
        const delay = Math.min(1000 * 2 ** (attempt - 1), 8000);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }, [url, maxRetries]);

  useEffect(() => {
    if (!url) return;
    const controller = new AbortController();
    fetchWithRetry(controller.signal);
    return () => controller.abort();
  }, [url, fetchWithRetry]);

  const retry = useCallback(() => {
    const controller = new AbortController();
    fetchWithRetry(controller.signal);
  }, [fetchWithRetry]);

  return { ...state, retry };
}

// Usage:
function UserProfile({ userId }: { userId: string }) {
  const { data, loading, error, attempt, retry } = useFetchWithRetry<User>(
    `/api/users/${userId}`
  );

  if (loading) return <Spinner label={`Attempt ${attempt}...`} />;
  if (error)   return <ErrorMessage error={error} onRetry={retry} />;
  return <Profile user={data} />;
}
```

</details>

---

## 🔗 Related Topics

- [`patterns/01-component-composition.md`](./01-component-composition.md) — Composing UI vs composing logic
- [`patterns/03-render-props-hoc.md`](./03-render-props-hoc.md) — Pre-hooks logic-sharing patterns
- [`testing/01-unit-testing.md`](../testing/01-unit-testing.md) — Testing strategies including hooks
- [`javascript-core/`](../javascript-core/) — Closures and refs that underlie hook mechanics

---

<div align="center">

**Next:** [`patterns/03-render-props-hoc.md`](./03-render-props-hoc.md) →

</div>
