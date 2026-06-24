# 04 — Stale Closures

> **"A stale closure is the gap between what a function remembered and what the world became. The function was born in one moment, captured what existed then, and is called in a different moment when those captured values have moved on — but the function doesn't know that. It still operates on the ghost of the past."**

Stale closures are one of the most subtle and persistent bug categories in React hooks. They occur when a function captures a value via closure at creation time, and that value later changes — but the function still uses the old, captured value because it was never updated. React's hooks model, with its dependency arrays and re-render-per-call semantics, creates abundant opportunities for stale closures to form silently and cause confusing bugs. This document covers how stale closures form, every common scenario in hooks, and the patterns that prevent them.

---

## 📚 Table of Contents

1. [How Closures Work in JavaScript](#1-how-closures-work-in-javascript)
2. [How React Hooks Create Stale Closure Opportunities](#2-how-react-hooks-create-stale-closure-opportunities)
3. [Stale Closures in useEffect](#3-stale-closures-in-useeffect)
4. [Stale Closures in useCallback](#4-stale-closures-in-usecallback)
5. [Stale Closures in setTimeout and setInterval](#5-stale-closures-in-settimeout-and-setinterval)
6. [Stale Closures in Event Listeners](#6-stale-closures-in-event-listeners)
7. [Stale Closures in Async Operations](#7-stale-closures-in-async-operations)
8. [Patterns to Prevent Stale Closures](#8-patterns-to-prevent-stale-closures)
9. [The useRef Solution](#9-the-useref-solution)
10. [Functional Updater Form of setState](#10-functional-updater-form-of-setstate)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. How Closures Work in JavaScript

```javascript
// A closure captures variables from its lexical scope at creation time
function createCounter(start) {
  let count = start; // captured by the returned function
  return function increment() {
    count++; // `count` is the captured variable
    return count;
  };
}

const counter = createCounter(0);
counter(); // 1
counter(); // 2
counter(); // 3
// Each call modifies the SAME `count` variable — no staleness here
// because `count` is a mutable variable, not a snapshot

// Staleness occurs when a closure captures a SNAPSHOT of a value
// that later changes INDEPENDENTLY of the closure
function createGreeter(name) {
  return function greet() {
    return `Hello, ${name}!`; // `name` is captured by value (strings are immutable)
  };
}

let userName = "Alice";
const greet = createGreeter(userName);
greet(); // "Hello, Alice!" ✓

userName = "Bob"; // name changed externally
greet(); // "Hello, Alice!" ← stale! greet still has the old `name`
// greet's captured `name` is the string "Alice" — a value snapshot, not a reference to `userName`
```

---

## 2. How React Hooks Create Stale Closure Opportunities

```jsx
// React components are called (rendered) as plain functions
// Each render: new function scope, new variable bindings
// Hooks (useEffect, useCallback, etc.) capture values from THAT render's scope

function Counter() {
  const [count, setCount] = useState(0); // each render: `count` is a new binding

  // Render 1: count = 0
  //   This useEffect closure captures: count = 0
  useEffect(() => {
    console.log(count); // logs 0 (from render 1's scope)
  }, []); // empty deps: only runs after render 1 — never sees count = 1, 2, 3...

  // Render 2: count = 1
  //   count is NOW 1 in this scope
  //   BUT the useEffect above still runs with count = 0 from render 1
  //   The empty deps array means "don't re-create this effect"

  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}

// KEY INSIGHT:
// React hooks capture a snapshot of values from the render they ran in.
// Hooks with dependency arrays only update when their deps change.
// If a captured value changes but is NOT in the deps array: staleness.
```

---

## 3. Stale Closures in useEffect

```jsx
// ❌ Classic stale closure in useEffect — missing dependency
function SearchComponent({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    const handler = setTimeout(() => {
      // `query` captured at effect creation time
      fetch(`/api/search?q=${query}`) // ← which `query`? The one from first render!
        .then((r) => r.json())
        .then(setResults);
    }, 300);

    return () => clearTimeout(handler);
  }, []); // ❌ empty deps: query changes but effect never re-runs with new query

  return <ResultsList results={results} />;
}

// Bug: user changes `query` prop → component re-renders with new `query`
// But the effect still holds a closure over the OLD `query` from first render
// The search always uses the initial query, ignoring subsequent changes

// ✅ Fix: add query to dependency array
useEffect(() => {
  const handler = setTimeout(() => {
    fetch(`/api/search?q=${query}`)
      .then((r) => r.json())
      .then(setResults);
  }, 300);
  return () => clearTimeout(handler);
}, [query]); // ✅ re-runs when query changes — always uses current query
```

### Subtle Stale Reference in useEffect

```jsx
// ❌ Object reference captured, but the object mutates
function Component({ user }) {
  const userRef = useRef(user);

  useEffect(() => {
    const interval = setInterval(() => {
      // `user` captured at first render — object reference may be stale
      logActivity(user.id);
    }, 5000);
    return () => clearInterval(interval);
  }, []); // ← user not in deps — always uses first render's user object
}

// If user prop changes (e.g., user logs in as a different account):
// The interval still logs activity for the ORIGINAL user
// Bug: incorrect user activity logging

// ✅ Include user in deps (or use ref pattern, Section 9)
useEffect(() => {
  const interval = setInterval(() => logActivity(user.id), 5000);
  return () => clearInterval(interval);
}, [user.id]); // re-creates interval when user.id changes
```

---

## 4. Stale Closures in useCallback

```jsx
// ❌ useCallback with missing dependency
function Parent({ items }) {
  const [filter, setFilter] = useState("");

  const getFilteredItems = useCallback(() => {
    // `filter` captured at time useCallback runs
    return items.filter((item) => item.name.includes(filter)); // stale `filter`!
  }, [items]); // ❌ `filter` missing from deps

  // User types in filter → filter state updates → component re-renders
  // BUT getFilteredItems still captures OLD filter (empty string from init)
  // Because `filter` not in deps, useCallback doesn't regenerate the function

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <ChildComponent getItems={getFilteredItems} />
    </>
  );
}

// ✅ Include all captured variables in deps
const getFilteredItems = useCallback(() => {
  return items.filter((item) => item.name.includes(filter));
}, [items, filter]); // ✅ both deps included — regenerates when either changes
```

---

## 5. Stale Closures in setTimeout and setInterval

```jsx
// ❌ setInterval captures initial count, never updates
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // count captured at effect creation: always 0!
      setCount(count + 1); // ❌ count is ALWAYS 0 here → setCount(0 + 1) = 1 forever
    }, 1000);
    return () => clearInterval(id);
  }, []); // empty deps: interval created once with count = 0

  return <div>{count}</div>;
  // Bug: count always shows 1 after first tick, never increments beyond that
}

// ✅ Solution 1: Include count in deps (recreates interval on each change — not ideal for intervals)
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]); // re-creates interval on each count change — works but inefficient

// ✅ Solution 2: Functional updater form (doesn't need count in closure)
useEffect(() => {
  const id = setInterval(() => {
    setCount((c) => c + 1); // ← uses previous state, no closure capture needed
  }, 1000);
  return () => clearInterval(id);
}, []); // ✅ empty deps is fine — functional updater has no closure over count

// ✅ Solution 3: useRef for interval with changing dependencies (Section 9)
```

---

## 6. Stale Closures in Event Listeners

```jsx
// ❌ Event listener added once, captures stale value
function Component() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [isTracking, setIsTracking] = useState(false);

  useEffect(() => {
    function handleMouseMove(e) {
      if (!isTracking) return; // ❌ `isTracking` captured at first render = false
      setPosition({ x: e.clientX, y: e.clientY });
    }

    document.addEventListener("mousemove", handleMouseMove);
    return () => document.removeEventListener("mousemove", handleMouseMove);
  }, []); // ❌ isTracking missing from deps

  // Bug: user toggles isTracking to true — component re-renders with isTracking=true
  // But handleMouseMove still has the OLD isTracking (false) captured
  // Mouse tracking never starts despite the state saying "tracking"

  return (
    <button onClick={() => setIsTracking((t) => !t)}>
      {isTracking ? "Stop" : "Start"} Tracking
    </button>
  );
}

// ✅ Fix: include isTracking in deps (re-registers listener on toggle)
useEffect(() => {
  function handleMouseMove(e) {
    if (!isTracking) return; // ← now always current value
    setPosition({ x: e.clientX, y: e.clientY });
  }
  document.addEventListener("mousemove", handleMouseMove);
  return () => document.removeEventListener("mousemove", handleMouseMove);
}, [isTracking]); // ✅ re-registers when isTracking changes

// ✅ Alternative: use ref (Section 9) to avoid re-registering the listener
```

---

## 7. Stale Closures in Async Operations

```jsx
// ❌ Async operation captures stale prop
function UserCard({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    async function fetchUser() {
      const result = await api.getUser(userId); // `userId` may be stale!
      setUser(result);
    }
    fetchUser();
  }, []); // ❌ userId missing from deps — always fetches first userId

  return user ? <div>{user.name}</div> : <Spinner />;
}

// Bug: userId prop changes (parent navigates to different user)
// The effect doesn't re-run (empty deps) → shows old user's data

// ✅ Fix 1: Include userId in deps + cancellation
useEffect(() => {
  const controller = new AbortController();

  api
    .getUser(userId, { signal: controller.signal })
    .then(setUser)
    .catch((err) => {
      if (err.name !== "AbortError") console.error(err);
    });

  return () => controller.abort(); // cancel if userId changes before fetch completes
}, [userId]); // ✅ re-fetches when userId changes
```

### Race Condition from Stale Closure

```jsx
// ❌ Async result from old request overwrites newer result
function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    // User types fast: queries "a", "ap", "app", "appl", "apple"
    // Five fetches start in parallel
    // If "app" fetch finishes AFTER "apple" fetch:
    //   → setResults called with "app" results after "apple" results
    //   → user sees wrong results for their current query!
    fetch(`/api/search?q=${query}`)
      .then((r) => r.json())
      .then(setResults); // ❌ always sets results, regardless of ordering
  }, [query]);
}

// ✅ Fix: ignore stale responses via cancellation flag
useEffect(() => {
  let cancelled = false;

  fetch(`/api/search?q=${query}`)
    .then((r) => r.json())
    .then((data) => {
      if (!cancelled) setResults(data); // ← only update if still the latest query
    });

  return () => {
    cancelled = true;
  }; // mark previous requests as stale
}, [query]);
// When query changes: cleanup runs (cancelled = true) → old fetch result ignored
// New fetch starts with new query — only its result reaches setResults
```

---

## 8. Patterns to Prevent Stale Closures

### Pattern 1: Complete Dependency Arrays

```jsx
// The foundational fix: include ALL captured variables in the deps array
// ESLint's react-hooks/exhaustive-deps rule enforces this automatically

// ✅ All deps declared
useEffect(() => {
  doSomething(a, b, c); // uses a, b, c
}, [a, b, c]); // ← all three in deps

// Use eslint-plugin-react-hooks:
// "react-hooks/exhaustive-deps": "warn" (or "error")
// This rule catches missing deps automatically
```

### Pattern 2: Stable Dependencies via Functional Updater

```jsx
// When you need previous state in an effect/callback: use functional updater
// so the function doesn't need to capture the current state value

// ❌ Captures count — needs count in deps, causes re-subscription
useEffect(() => {
  socket.on("message", () => setCount(count + 1)); // needs count
  return () => socket.off("message");
}, [count]); // re-subscribes every time count changes — undesirable

// ✅ Functional updater: doesn't capture count at all
useEffect(() => {
  socket.on("message", () => setCount((c) => c + 1)); // no capture of count
  return () => socket.off("message");
}, []); // empty deps is correct — no stale closure risk
```

### Pattern 3: Derive Inside the Effect

```jsx
// ❌ Capturing a derived value that may be stale
const processedData = useMemo(() => transform(rawData), [rawData]);

useEffect(() => {
  doSomethingWith(processedData); // captures the memo's cached value
}, [someOtherDep]); // processedData might be stale if rawData changed

// ✅ Re-derive inside the effect (no capture of derived value)
useEffect(() => {
  const processedData = transform(rawData); // always current
  doSomethingWith(processedData);
}, [rawData, someOtherDep]);
```

---

## 9. The useRef Solution

`useRef` creates a mutable container that persists across renders. Storing values in refs lets effects always access the latest value without including them in the dependency array:

```jsx
// Pattern: store latest value in ref — effects always read current value
function useLatest<T>(value: T): React.MutableRefObject<T> {
  const ref = useRef(value);
  ref.current = value; // update synchronously on every render
  return ref;
}

// Usage: event listener that always sees current callback
function useEventListener(
  event: string,
  handler: (e: Event) => void,
  element: Window | HTMLElement = window
) {
  const savedHandler = useLatest(handler);
  // savedHandler.current is ALWAYS the latest handler

  useEffect(() => {
    const listener = (e: Event) => savedHandler.current(e);
    element.addEventListener(event, listener);
    return () => element.removeEventListener(event, listener);
  }, [event, element]); // ← savedHandler NOT in deps (it's a ref — stable reference)
  // The listener always calls the LATEST handler via the ref
  // No stale closure: handler is read from ref at call time, not captured at creation time
}

// Usage: no dependency on handler — can be an inline arrow function
function Component() {
  const [count, setCount] = useState(0);

  useEventListener('keydown', (e) => {
    if (e.key === 'ArrowUp') setCount(c => c + 1);
    console.log('current count:', count); // ← always current count via closure over state
  });
  // Handler inline arrow — new function every render
  // Without useLatest: would need to add handler to deps → re-register on every render
  // With useLatest: handler ref always current → no re-registration needed
}
```

### The useRef Pattern for setInterval

```jsx
// Interval that always runs the latest callback without recreating the interval
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);
  savedCallback.current = callback; // always the latest

  useEffect(() => {
    if (delay === null) return; // null = paused

    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]); // only recreate interval if delay changes, not callback
}

// Usage: callback can reference any state — no stale closure
function Timer() {
  const [count, setCount] = useState(0);
  const [multiplier, setMultiplier] = useState(1);

  useInterval(() => {
    // This always sees current count and multiplier — no stale closure
    setCount(c => c + multiplier); // uses current multiplier via closure
  }, 1000);

  return <div>{count}</div>;
}
```

---

## 10. Functional Updater Form of setState

The functional updater form `setState(prev => next)` avoids stale closures by not needing to capture current state:

```jsx
// ❌ Direct update: needs to capture current state
function CounterWithSocketBug() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    socket.on("increment", () => {
      setCount(count + 1); // ❌ count captured at effect creation = always 0
    });
    return () => socket.off("increment");
  }, []); // count not in deps — stale closure
}

// ✅ Functional updater: no need to capture count
function CounterWithSocket() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    socket.on("increment", () => {
      setCount((c) => c + 1); // ✅ c is ALWAYS the latest state — provided by React
    });
    return () => socket.off("increment");
  }, []); // empty deps correct — functional updater has no stale closure risk
}

// Works for any derived update:
setItems((prev) => [...prev, newItem]);
setSet((prev) => new Set([...prev, newId]));
setMap((prev) => new Map([...prev, [key, value]]));
```

### When Functional Updater Isn't Enough

```jsx
// Functional updater only helps for setState
// If you need OTHER stale values in the effect: use deps array or useRef

function Component() {
  const [count, setCount] = useState(0);
  const [userId, setUserId] = useState(null);

  useEffect(() => {
    socket.on("score", (delta) => {
      // Functional updater for count ✓
      // But also needs current userId — functional updater can't help with this
      setCount((c) => c + delta);
      logScore(userId, delta); // ← userId is captured at effect creation!
    });
    return () => socket.off("score");
  }, []); // userId not in deps — stale!
}

// ✅ Fix: either add userId to deps, or use useRef
const userIdRef = useRef(userId);
userIdRef.current = userId;

useEffect(() => {
  socket.on("score", (delta) => {
    setCount((c) => c + delta); // functional updater ✓
    logScore(userIdRef.current, delta); // ref always current ✓
  });
  return () => socket.off("score");
}, []); // both issues resolved
```

---

## 11. Good Practices

### ✅ Enable ESLint exhaustive-deps rule

```json
// .eslintrc
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
// This catches missing dependencies at lint time — before runtime
```

### ✅ Use functional updater form for all state updates in effects

```jsx
// ✅ Default to functional updater — avoids entire class of stale closure bugs
setCount((c) => c + 1);
setItems((prev) => [...prev, item]);
setError(null); // for non-derived updates: direct form is fine
```

### ✅ Use useLatest for callbacks in long-lived effects

```jsx
// ✅ Wrap callbacks that change frequently in a ref for event listeners and intervals
const savedCallback = useRef(callback);
savedCallback.current = callback;
```

---

## 12. Bad Practices

### ❌ Suppressing the exhaustive-deps warning without understanding why

```jsx
// ❌ Silencing the warning instead of fixing the deps
useEffect(() => {
  doSomething(value);
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []); // suppressed warning = hidden stale closure

// ✅ Either add value to deps, or use the ref pattern if re-running is undesirable
```

### ❌ Adding a function to deps that's recreated every render

```jsx
// ❌ `onUpdate` is new every render → effect re-runs on EVERY render
useEffect(() => {
  doWork(onUpdate);
}, [onUpdate]); // onUpdate is an inline prop function: () => handleUpdate(id)

// ✅ Stabilize the function first with useCallback in the parent
const handleUpdate = useCallback(() => update(id), [id]);
// OR: use useLatest/useRef pattern to avoid including it in deps
```

---

## 13. Common Mistakes

### Mistake 1 — Empty deps array on a fetch effect

```jsx
// ❌ The most common stale closure bug: fetch with empty deps
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then(setUser);
  }, []); // ❌ userId changes → effect doesn't re-run → shows wrong user!

  // ✅ Always include the variable the URL depends on
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then(setUser);
  }, [userId]);
}
```

### Mistake 2 — Object identity in deps array

```jsx
// ❌ Object reference is new every render → effect re-runs endlessly
useEffect(() => {
  doSomething(options);
}, [options]); // `options` = { a: 1 } created inline → new object every render!

// ✅ Use primitive values or memoized objects in deps
useEffect(() => {
  doSomething(options);
}, [options.a, options.b]); // use the primitive VALUES from the object

// OR: memoize the options object
const stableOptions = useMemo(() => ({ a, b }), [a, b]);
useEffect(() => {
  doSomething(stableOptions);
}, [stableOptions]);
```

### Mistake 3 — Assuming re-renders update closure values

```jsx
// ❌ Thinking that because the component re-rendered, the closure is updated
function Component() {
  const [count, setCount] = useState(0);

  const logCount = () => {
    setTimeout(() => {
      console.log(count); // ← captures count from CURRENT render only
    }, 2000);
  };
  // If user clicks "Log", then increments count before 2 seconds:
  // The log will show the OLD count from when the button was clicked, not the current count

  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <button onClick={logCount}>Log in 2s</button>
      <p>{count}</p>
    </>
  );
}
// This is not a bug per se — it's often intended (log the count AT THE TIME of click)
// But if you want the CURRENT count: use a ref
```

---

## 14. Interview-Level Explanation

> **"What is a stale closure in React? How do you prevent them?"**

**Strong answer:**

> "A stale closure occurs when a function captures a value from its surrounding scope at creation time, and that value later changes independently — but the function still uses the old captured value because it was never updated.
>
> In React, this is particularly insidious with hooks because every render creates a new scope with new variable bindings, and hooks like `useEffect` and `useCallback` create closures over values from the render they run in. If a `useEffect` has an empty dependency array, it only runs once and its closure captures values from the first render only. When those values change on subsequent renders, the effect still sees the original values — classic stale closure.
>
> The four main contexts where stale closures appear: `useEffect` with missing dependencies (a fetch that always uses the first render's props, an event handler that always evaluates the initial state), `useCallback` with missing dependencies (a callback function that doesn't reflect current state), `setInterval` and `setTimeout` (timers capture state at creation and never update), and async operations (a fetch that completes after the component has received new props and updates state with data for the old props, causing a race condition).
>
> The primary fix is the exhaustive-deps ESLint rule — it catches missing dependencies at lint time. For each missing dep, the fix is either to add it to the array, switch to a functional updater form of setState (which doesn't need to capture current state), or use the ref pattern.
>
> The ref pattern is the most versatile solution for long-lived effects: `const savedRef = useRef(value); savedRef.current = value;` The ref object is stable across renders, so it can be safely omitted from deps. But unlike a captured variable, reading `savedRef.current` at call time always gives the latest value. This is the right pattern for event listeners and intervals that should always see the current callback without being re-registered on every state change.
>
> The functional updater form `setState(prev => next)` eliminates stale closures for state updates: the updater receives the current state as an argument rather than capturing it via closure. An interval that calls `setCount(c => c + 1)` never needs `count` in its closure — React injects the current state at call time."

---

## 15. Exercises

### Exercise 1 — Find and fix the stale closures

```jsx
// Identify ALL stale closure bugs in this component and fix each one.

function ChatRoom({ roomId, username }) {
  const [messages, setMessages] = useState([]);
  const [typingStatus, setTypingStatus] = useState('');

  // Bug 1
  useEffect(() => {
    socket.emit('join', roomId);
    socket.on('message', (msg) => {
      setMessages([...messages, msg]); // depends on messages
    });
    return () => {
      socket.emit('leave', roomId);
      socket.off('message');
    };
  }, []); // ← deps

  // Bug 2
  const sendMessage = useCallback((text) => {
    socket.emit('send', { room: roomId, user: username, text });
  }, []); // ← deps

  // Bug 3
  useEffect(() => {
    const interval = setInterval(() => {
      if (messages.length > 0) { // depends on messages
        socket.emit('heartbeat', { room: roomId, user: username });
      }
    }, 10_000);
    return () => clearInterval(interval);
  }, []); // ← deps

  return (/* ... */);
}
```

<details>
<summary>Solution</summary>

```jsx
function ChatRoom({ roomId, username }) {
  const [messages, setMessages] = useState([]);
  const [typingStatus, setTypingStatus] = useState('');

  // FIX 1: setMessages with functional updater avoids capturing `messages`
  // roomId added to deps so effect re-runs when room changes
  useEffect(() => {
    socket.emit('join', roomId);

    socket.on('message', (msg) => {
      setMessages(prev => [...prev, msg]); // ✅ functional updater — no stale `messages`
    });

    return () => {
      socket.emit('leave', roomId);
      socket.off('message');
    };
  }, [roomId]); // ✅ re-runs when roomId changes (joins new room, leaves old)

  // FIX 2: roomId and username both captured — include both in deps
  const sendMessage = useCallback((text) => {
    socket.emit('send', { room: roomId, user: username, text });
  }, [roomId, username]); // ✅ regenerates when either changes

  // FIX 3: messages.length and roomId/username captured in interval callback
  // Use ref pattern to avoid re-creating the interval on every message
  const latestRef = useRef({ messages, roomId, username });
  latestRef.current = { messages, roomId, username }; // always current

  useEffect(() => {
    const interval = setInterval(() => {
      const { messages, roomId, username } = latestRef.current; // ✅ always current via ref
      if (messages.length > 0) {
        socket.emit('heartbeat', { room: roomId, user: username });
      }
    }, 10_000);
    return () => clearInterval(interval);
  }, []); // ✅ empty deps correct — ref provides current values without closure capture

  return (/* ... */);
}

// SUMMARY OF FIXES:
// Bug 1: setMessages([...messages, msg]) → setMessages(prev => [...prev, msg])
//         roomId missing from deps → add roomId
// Bug 2: roomId and username not in deps → add both
// Bug 3: messages, roomId, username all missing from deps
//         Solution: store in ref, read current from ref inside interval
//         (Adding them to deps would cause interval to reset on every message — bad UX)
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/04-closures.md`](../javascript-core/04-closures.md) — Closures fundamentals
- [`anti-patterns/03-premature-optimization.md`](./03-premature-optimization.md) — useCallback/useMemo pitfalls
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md) — Hook design that avoids stale closures

---

<div align="center">

**Next:** [`anti-patterns/05-memory-leaks.md`](./05-memory-leaks.md) →

</div>
