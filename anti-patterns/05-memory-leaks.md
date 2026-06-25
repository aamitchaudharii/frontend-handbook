# 05 — Memory Leaks

> **"A memory leak is a silent promise you forgot to keep. You said 'I'll stop listening when you're gone' but you never stopped. You said 'I'll cancel that request when I'm done' but you never cancelled. The garbage collector is waiting for you to let go, and you never do."**

Memory leaks in frontend applications cause pages to become progressively slower, consume more RAM, and eventually crash the browser tab. Unlike backend server leaks which fail loudly and quickly, frontend leaks can run silently for hours — the user just notices the app slowing down over time and refreshes the page. This document covers every common source of memory leaks in React and JavaScript: abandoned event listeners, uncancelled async operations, lingering subscriptions, detached DOM nodes, closures holding large objects, and the profiling techniques to find leaks that already exist.

---

## 📚 Table of Contents

1. [What Causes Memory Leaks](#1-what-causes-memory-leaks)
2. [Leak 1 — Missing useEffect Cleanup](#2-leak-1--missing-useeffect-cleanup)
3. [Leak 2 — Event Listeners Without Removal](#3-leak-2--event-listeners-without-removal)
4. [Leak 3 — Uncancelled Async Operations](#4-leak-3--uncancelled-async-operations)
5. [Leak 4 — WebSocket and SSE Connections](#5-leak-4--websocket-and-sse-connections)
6. [Leak 5 — Timers and Intervals](#6-leak-5--timers-and-intervals)
7. [Leak 6 — Closures Capturing Large Objects](#7-leak-6--closures-capturing-large-objects)
8. [Leak 7 — Detached DOM Nodes](#8-leak-7--detached-dom-nodes)
9. [Leak 8 — Global Caches Without Eviction](#9-leak-8--global-caches-without-eviction)
10. [Profiling Memory Leaks in DevTools](#10-profiling-memory-leaks-in-devtools)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. What Causes Memory Leaks

```
GARBAGE COLLECTION:
  JavaScript's GC automatically frees memory when objects have no remaining references.
  A memory leak occurs when an object is still REFERENCED but is no longer NEEDED.
  The GC cannot collect it because something is still pointing to it.

COMMON LEAK ROOT CAUSES:

  UNMOUNTED COMPONENT STILL REFERENCED BY:
    - Event listener on window/document pointing to component callback
    - setInterval/setTimeout callback closing over component state
    - WebSocket/EventSource with onmessage pointing to component handler
    - Promise resolve/reject calling component setState
    - Global store listener holding component callback
    - Third-party library callback not cleaned up

  DETACHED DOM NODES:
    - Reference to DOM node stored in JS variable
    - Node removed from DOM but JS variable still holds it
    - DOM subtree not GC'd because one node is referenced

  INFINITE ACCUMULATION:
    - Global Map/Set/Array that grows without bound
    - Event system that allows duplicate listener registration
    - Cache without eviction policy
```

---

## 2. Leak 1 — Missing useEffect Cleanup

```jsx
// ❌ Classic: subscription created in useEffect, never cleaned up
function UserList() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    // Subscribe to real-time user updates
    const subscription = userStore.subscribe((newUsers) => {
      setUsers(newUsers); // ← this callback holds a reference to the component
    });
    // NO cleanup function returned!
    // When UserList unmounts: subscription persists
    // userStore still holds the callback → component state/closures cannot be GC'd
    // If UserList mounts/unmounts repeatedly: multiple subscriptions accumulate
  }, []);

  return users.map((u) => <UserCard key={u.id} user={u} />);
}

// ✅ Always return a cleanup function from effects with subscriptions
useEffect(() => {
  const subscription = userStore.subscribe(setUsers);
  return () => subscription.unsubscribe(); // ← cleanup: removes the reference
}, []);
// When UserList unmounts: cleanup runs, subscription removed
// userStore no longer holds the callback → component can be GC'd
```

### Every Effect That Creates Something Must Clean It Up

```jsx
// Pattern: every resource created in useEffect has a corresponding cleanup

// ✅ Interval
useEffect(() => {
  const id = setInterval(poll, 5000);
  return () => clearInterval(id);
}, []);

// ✅ Timeout
useEffect(() => {
  const id = setTimeout(callback, 1000);
  return () => clearTimeout(id);
}, []);

// ✅ Event listener
useEffect(() => {
  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize);
}, [handleResize]);

// ✅ Observer
useEffect(() => {
  const observer = new IntersectionObserver(callback);
  observer.observe(ref.current);
  return () => observer.disconnect();
}, []);

// ✅ WebSocket
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = handleMessage;
  return () => ws.close();
}, [url]);

// ✅ Third-party library
useEffect(() => {
  const map = initializeMapLibrary(containerRef.current);
  return () => map.destroy(); // library's cleanup method
}, []);
```

---

## 3. Leak 2 — Event Listeners Without Removal

```javascript
// ❌ Event listener added to DOM but never removed
document.addEventListener("keydown", function handler(e) {
  if (e.key === "Escape") closeModal();
});
// This handler persists FOREVER — it's never removed
// If this code runs on every page navigation: accumulates more handlers
// Memory: handler function, its closure (closeModal reference), cannot be GC'd

// ❌ React component adding global listeners without cleanup
function Modal({ onClose }) {
  useEffect(() => {
    document.addEventListener("keydown", (e) => {
      if (e.key === "Escape") onClose(); // new anonymous function every render!
    });
    // No cleanup — handler leaks on unmount
    // Plus: if effect re-runs (onClose changes), ANOTHER handler is added
    // Multiple handlers firing per keydown event!
  }); // no deps array: runs on every render!
}

// ✅ Correct: named function + cleanup
function Modal({ onClose }) {
  useEffect(() => {
    function handleKeyDown(e) {
      if (e.key === "Escape") onClose();
    }
    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [onClose]); // onClose in deps: re-registers if onClose changes
}
```

### Passive Listener Accumulation

```javascript
// ❌ Multiple registrations of the same listener (common pattern bug)
class EventBus {
  constructor() {
    this.listeners = {};
  }

  on(event, handler) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(handler); // ← ALWAYS adds, never checks for duplicates
  }
}

// If a React component registers in every render:
useEffect(() => {
  eventBus.on("update", handleUpdate); // adds a NEW handler each render
}); // no deps array: runs on every render!
// After 100 renders: 100 handlers firing for each event

// ✅ Deduplicate OR always clean up
useEffect(() => {
  eventBus.on("update", handleUpdate);
  return () => eventBus.off("update", handleUpdate); // remove on cleanup
}, [handleUpdate]); // stable handleUpdate reference (useCallback)
```

---

## 4. Leak 3 — Uncancelled Async Operations

```jsx
// ❌ setState called after unmount (React 18+ shows a warning, pre-18 silently leaks)
function ProductDetail({ productId }) {
  const [product, setProduct] = useState(null);

  useEffect(() => {
    fetch(`/api/products/${productId}`)
      .then((r) => r.json())
      .then((data) => {
        setProduct(data); // ← called after unmount if component unmounts during fetch!
        // Pre-React 18: "Warning: Can't perform a React state update on an unmounted component"
        // The closure keeps the component instance alive until the promise resolves
      });
    // No cleanup → if productId changes: old fetch and new fetch race
  }, [productId]);
}

// ✅ Fix: AbortController cancels the fetch, prevents the callback from firing
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/products/${productId}`, { signal: controller.signal })
    .then((r) => r.json())
    .then(setProduct)
    .catch((err) => {
      if (err.name !== "AbortError") console.error(err); // ignore abort errors
    });

  return () => controller.abort(); // cancel on unmount or productId change
}, [productId]);
```

### Promises That Can't Be Cancelled

```javascript
// When you can't use AbortController (non-fetch async):
// Use a cancellation flag

function useAsync(asyncFn, deps) {
  const [state, setState] = useState({
    loading: true,
    data: null,
    error: null,
  });

  useEffect(() => {
    let cancelled = false; // ← cancellation flag

    setState({ loading: true, data: null, error: null });

    asyncFn()
      .then((data) => {
        if (!cancelled) setState({ loading: false, data, error: null });
        // If cancelled: setState is NOT called — no state update on unmounted component
      })
      .catch((error) => {
        if (!cancelled) setState({ loading: false, data: null, error });
      });

    return () => {
      cancelled = true;
    }; // set flag on cleanup
  }, deps);

  return state;
}
```

---

## 5. Leak 4 — WebSocket and SSE Connections

```jsx
// ❌ WebSocket not closed on unmount — connection persists forever
function LiveFeed({ feedId }) {
  const [items, setItems] = useState([]);

  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/feeds/${feedId}`);
    ws.onmessage = (e) => setItems((prev) => [...prev, JSON.parse(e.data)]);
    // No cleanup!
    // After unmount: ws still connected, still receiving messages
    // ws.onmessage closure still references setItems and the component
    // Component cannot be GC'd while ws exists and references it
  }, [feedId]);
}

// ✅ Close WebSocket on cleanup
useEffect(() => {
  const ws = new WebSocket(`wss://api.example.com/feeds/${feedId}`);
  ws.onmessage = (e) => setItems((prev) => [...prev, JSON.parse(e.data)]);

  return () => {
    ws.onmessage = null; // remove handler first
    ws.close(); // close connection
  };
}, [feedId]);

// ❌ EventSource (SSE) not closed on unmount
useEffect(() => {
  const source = new EventSource(`/api/events?room=${roomId}`);
  source.onmessage = handleEvent;
  // No cleanup!
}, [roomId]);

// ✅ Close SSE on cleanup
useEffect(() => {
  const source = new EventSource(`/api/events?room=${roomId}`);
  source.onmessage = handleEvent;
  return () => source.close();
}, [roomId]);
```

---

## 6. Leak 5 — Timers and Intervals

```jsx
// ❌ setInterval not cleared on unmount
function AutoSave({ content }) {
  useEffect(() => {
    const interval = setInterval(() => {
      saveToServer(content); // runs every 30s forever
    }, 30_000);
    // No cleanup! Interval persists after unmount.
    // content is captured in the closure → component GC'd? No, interval holds it
  }, [content]);
}

// ✅ Clear interval in cleanup
useEffect(() => {
  const interval = setInterval(() => {
    saveToServer(content);
  }, 30_000);
  return () => clearInterval(interval);
}, [content]);

// ❌ Timeout in event handler — can't be cancelled if component unmounts
function NotificationBanner() {
  const [visible, setVisible] = useState(true);

  function handleClick() {
    // User clicks "dismiss" — triggers auto-hide
    setTimeout(() => setVisible(false), 300); // if component unmounts: leaks
  }
  // ...
}

// ✅ Track and clear the timeout
function NotificationBanner() {
  const [visible, setVisible] = useState(true);
  const timeoutRef = useRef(null);

  function handleClick() {
    timeoutRef.current = setTimeout(() => setVisible(false), 300);
  }

  useEffect(() => {
    return () => {
      if (timeoutRef.current) clearTimeout(timeoutRef.current);
    };
  }, []);
}
```

---

## 7. Leak 6 — Closures Capturing Large Objects

```javascript
// ❌ Closure holds reference to large object long after it's needed
function setupAnalytics(largeDataset) {
  const processed = process(largeDataset); // large intermediate object

  return function trackEvent(event) {
    // trackEvent only needs `processed.id`, not the whole object
    // But it captures `processed` in its closure
    // `largeDataset` is also kept alive (closure chain)
    sendEvent({ event, datasetId: processed.id });
  };
}

const tracker = setupAnalytics(hugeDataset); // hugeDataset cannot be GC'd
// hugeDataset: 50MB, only .id needed. 50MB leak.

// ✅ Extract only what's needed to break the closure chain
function setupAnalytics(largeDataset) {
  const datasetId = process(largeDataset).id; // extract only the needed value
  // largeDataset and processed are now eligible for GC — nothing captures them

  return function trackEvent(event) {
    sendEvent({ event, datasetId }); // only captures the primitive `datasetId`
  };
}
```

### React: Closures in useCallback Capturing Entire Objects

```jsx
// ❌ useCallback captures full user object when only id is needed
function UserList({ users }) {
  const handleDelete = useCallback(
    (id) => {
      const user = users.find((u) => u.id === id);
      // Uses only user.id but closes over entire `users` array (potentially large)
      archiveUser(user.id); // only needs the id!
    },
    [users],
  ); // captures entire users array in closure
}

// ✅ Pass only needed data; don't capture large collections unnecessarily
function UserList({ users }) {
  // Pass the id directly to the handler — no need to capture `users`
  return users.map((user) => (
    <UserRow key={user.id} user={user} onDelete={archiveUser} />
  ));
  // UserRow receives the handler and its own user — no closure over `users`
}
```

---

## 8. Leak 7 — Detached DOM Nodes

```javascript
// ❌ Reference to a removed DOM node prevents GC of entire subtree
const listContainer = document.getElementById("list");
const items = {}; // stores references to DOM nodes

function addItem(id, element) {
  listContainer.appendChild(element);
  items[id] = element; // store reference to DOM node
}

function removeItem(id) {
  items[id].remove(); // removes from DOM
  // items[id] still holds the reference! DOM node is "detached"
  // The detached node's entire subtree cannot be GC'd
  // If the element had 100 child nodes: all 100+ stay in memory
}

// ✅ Remove the JavaScript reference when removing from DOM
function removeItem(id) {
  items[id].remove();
  delete items[id]; // ← release the reference → node is eligible for GC
}
```

```jsx
// React: refs that are not cleared can hold detached DOM nodes
function Component() {
  const nodeRef = useRef(null);

  // If nodeRef.current is used outside this component (stored in a global),
  // the DOM node will stay in memory even after the component unmounts

  // ✅ React cleans up refs automatically on unmount
  // The concern: if YOU store ref.current somewhere external:
  someGlobalCache.set("myNode", nodeRef.current); // ← this leaks!
  // Clear external references in useEffect cleanup
  useEffect(() => {
    someGlobalCache.set("myNode", nodeRef.current);
    return () => someGlobalCache.delete("myNode"); // clean up on unmount
  }, []);
}
```

---

## 9. Leak 8 — Global Caches Without Eviction

```javascript
// ❌ Global Map that grows without bound
const imageCache = new Map(); // module-level cache

function loadImage(url) {
  if (imageCache.has(url)) return imageCache.get(url);
  const img = new Image();
  img.src = url;
  imageCache.set(url, img); // stored forever — no eviction!
  return img;
}
// After loading 10,000 unique images: 10,000 Image objects in memory
// Never GC'd because imageCache holds all references

// ✅ Bounded cache with LRU eviction
const imageCache = new LRUCache(200); // max 200 images

// ✅ WeakRef for cache entries (GC'd under memory pressure)
const weakCache = new Map(); // Map of url → WeakRef<Image>

function loadImage(url) {
  const ref = weakCache.get(url);
  if (ref) {
    const cached = ref.deref();
    if (cached) return cached; // still in memory
    weakCache.delete(url); // GC'd — remove dead entry
  }
  const img = new Image();
  img.src = url;
  weakCache.set(url, new WeakRef(img)); // weak reference: can be GC'd
  return img;
}
```

---

## 10. Profiling Memory Leaks in DevTools

### Chrome DevTools Memory Panel

```
1. DETECT GROWTH OVER TIME (Timeline):
   DevTools → Memory → Allocation instrumentation on timeline → Start
   Perform the suspected leaking action (e.g., navigate between pages)
   Stop → look for blue bars that don't get reclaimed (grey)
   Blue = allocated, grey = reclaimed after GC
   Persistent blue bars = potential leak

2. FIND THE LEAKING OBJECT (Heap Snapshot):
   DevTools → Memory → Heap snapshot → Take snapshot
   Perform leaking action several times
   Take second snapshot
   Select snapshot 2 → "Comparison" view → sort by "# Delta"
   Objects increasing in count between snapshots = potential leak
   Click an object → "Retainers" shows WHY it's still referenced

3. DETECT DETACHED DOM NODES:
   Heap snapshot → filter by "Detached"
   Shows all DOM nodes removed from the tree but still referenced in JS
   Click to see which JS code holds the reference

4. QUICK LEAK CHECK — The three-snapshot technique:
   Snapshot 1 (baseline)
   Perform action (mount/unmount component 5 times)
   Click GC button (trash can icon in Memory panel)
   Snapshot 2
   Perform action again (5 more times)
   Click GC
   Snapshot 3
   Compare 1→2 and 2→3: if growing consistently = leak
```

### Performance Monitor for Ongoing Tracking

```javascript
// Chrome DevTools → More tools → Performance Monitor
// Watch these metrics in real time:
// - JS heap size: should plateau, not grow continuously
// - DOM Nodes: should stay stable, not grow indefinitely
// - JS event listeners: should not grow with navigation

// Automated memory monitoring in development
if (process.env.NODE_ENV === "development") {
  setInterval(() => {
    if (performance.memory) {
      const { usedJSHeapSize, jsHeapSizeLimit } = performance.memory;
      const usedPercent = ((usedJSHeapSize / jsHeapSizeLimit) * 100).toFixed(1);
      if (usedPercent > 80) {
        console.warn(
          `⚠️ High JS heap usage: ${usedPercent}% (${(usedJSHeapSize / 1e6).toFixed(1)}MB)`,
        );
      }
    }
  }, 10_000);
}
```

---

## 11. Good Practices

### ✅ Always return cleanup from effects that create subscriptions, listeners, or connections

```jsx
// ✅ Systematic: every effect follows the create + cleanup pattern
useEffect(() => {
  const sub = store.subscribe(handleChange);
  return () => sub.unsubscribe(); // always present
}, []);
```

### ✅ Use AbortController for all fetch operations inside effects

```jsx
// ✅ Cancel in-flight requests when component unmounts or deps change
useEffect(() => {
  const ctrl = new AbortController();
  fetchData(url, { signal: ctrl.signal })
    .then(setData)
    .catch(() => {});
  return () => ctrl.abort();
}, [url]);
```

### ✅ Bound all in-memory caches

```javascript
// ✅ All caches have a maximum size
const cache = new LRUCache(500);
// Or: use TTL-based eviction
const cache = new TTLCache(60_000); // entries expire after 60 seconds
```

### ✅ Use WeakMap for metadata on objects you don't own

```javascript
// ✅ WeakMap: metadata automatically GC'd when the key object is GC'd
const nodeMetadata = new WeakMap(); // DOM node → metadata
nodeMetadata.set(element, { tooltip: "Click me" });
// When element is removed from DOM and GC'd: this entry disappears automatically
```

---

## 12. Bad Practices

### ❌ Storing large objects in module-level variables without cleanup

```javascript
// ❌ Module-level variable accumulates state forever
const processedReports = []; // grows throughout the session

function processReport(report) {
  processedReports.push(heavyTransform(report)); // never cleared
}
// After 1 hour of use: processedReports holds hours of transformed data
```

### ❌ Unconditional global listeners in component files

```javascript
// ❌ Listener added at module level, lives forever
window.addEventListener("resize", handleResize); // module-level, never removed

export function MyComponent() {
  /* ... */
}
// handleResize runs forever even when MyComponent is not rendered
```

---

## 13. Common Mistakes

### Mistake 1 — Not understanding that closures keep entire scopes alive

```javascript
// ❌ Heavy computation result kept alive by a tiny closure
function createButtonHandler() {
  const massiveData = loadMassiveDataset(); // 100MB object
  const summary = computeSummary(massiveData); // small object

  return function handleClick() {
    displaySummary(summary); // only uses `summary` — but closes over the entire scope!
    // `massiveData` is in the enclosing scope → still referenced → NOT GC'd
  };
}

// ✅ Extract only what's needed to break the reference
function createButtonHandler() {
  const massiveData = loadMassiveDataset();
  const summary = computeSummary(massiveData);
  // massiveData eligible for GC after this point — nothing references it

  const summaryRef = { ...summary }; // extract to new object
  // massiveData is no longer reachable from anything that survives

  return function handleClick() {
    displaySummary(summaryRef); // closes over summaryRef, not the original scope
  };
}
```

### Mistake 2 — Missing cleanup in custom hooks

```javascript
// ❌ Custom hook that creates a subscription but doesn't clean it up
function useRealTimeData(channel) {
  const [data, setData] = useState(null);

  useEffect(() => {
    realtimeClient.subscribe(channel, setData); // subscription created
    // No cleanup returned from the hook!
    // Every component using this hook leaks subscriptions on unmount
  }, [channel]);

  return data;
}

// ✅ Custom hooks must clean up effects they create
function useRealTimeData(channel) {
  const [data, setData] = useState(null);

  useEffect(() => {
    const sub = realtimeClient.subscribe(channel, setData);
    return () => sub.unsubscribe(); // cleanup
  }, [channel]);

  return data;
}
```

### Mistake 3 — React.createRef in class component rendering

```jsx
// ❌ new ref created every render in class component
class Component extends React.Component {
  render() {
    const ref = React.createRef(); // new ref object every render!
    // Previous ref object becomes orphaned — can't be GC'd if something holds it
    return <div ref={ref}>...</div>;
  }
}

// ✅ Ref created once in constructor
class Component extends React.Component {
  constructor(props) {
    super(props);
    this.ref = React.createRef(); // one ref, reused across renders
  }
  render() {
    return <div ref={this.ref}>...</div>;
  }
}
```

---

## 14. Interview-Level Explanation

> **"What causes memory leaks in React applications? How do you find and fix them?"**

**Strong answer:**

> "Memory leaks in React occur when objects remain referenced in JavaScript even after they're no longer needed. The garbage collector can't collect something as long as something else points to it — a leak is a reference you forgot to remove.
>
> The most common sources: uncleared subscriptions in `useEffect` — you subscribe to a store or WebSocket in an effect, the component unmounts, but the subscription still holds a callback that references the component's state and closures. The component instance can't be GC'd. The fix is always returning a cleanup function from `useEffect` that removes the subscription. Every effect that creates a timer, event listener, observer, subscription, or network connection needs a corresponding cleanup.
>
> Uncancelled async operations are related: if a component fetches data and unmounts before the fetch completes, the fetch callback still holds a closure over `setState`, preventing the component from being GC'd. The fix is `AbortController` — abort the fetch in the effect's cleanup function. For non-fetch async operations, a cancellation flag (`let cancelled = false; ... if (!cancelled) setState(...)`) achieves the same result.
>
> Global caches without eviction are a subtler leak: a `Map` at module level that accumulates data throughout the session without any eviction policy. The fix is bounding it — an LRU cache with a maximum entry count, or a TTL cache that expires old entries.
>
> Closures capturing larger scopes than necessary can cause surprising leaks. A callback that only needs one field of a large object will keep the entire object alive because closures capture their lexical scope. The fix is extracting just the needed value before creating the closure, so the large object is no longer reachable through the closure chain.
>
> For profiling: Chrome DevTools Memory panel has three key techniques. The allocation timeline shows memory growing over time with blue bars for allocations and grey for collected. The heap snapshot shows all live objects, and the comparison view between two snapshots shows what was created and not collected between them. The retainer tree for any growing object shows exactly which reference path keeps it alive. Filtering heap snapshots for 'Detached' shows DOM nodes removed from the tree but still referenced in JavaScript."

---

## 15. Exercises

### Exercise 1 — Identify all memory leaks

```jsx
// This component has multiple memory leaks. Find and fix all of them.

function RealTimeDashboard({ roomId, userId }) {
  const [messages, setMessages] = useState([]);
  const [onlineUsers, setOnlineUsers] = useState([]);
  const [metrics, setMetrics] = useState({});
  const reportCache = {}; // module-level cache (imagine this is outside the component)

  useEffect(() => {
    // Leak A
    const ws = new WebSocket(`wss://api.example.com/room/${roomId}`);
    ws.onmessage = (e) => setMessages(prev => [...prev, JSON.parse(e.data)]);
  }, [roomId]);

  useEffect(() => {
    // Leak B
    window.addEventListener('focus', () => {
      fetchLatestMessages(roomId).then(setMessages);
    });
  }, [roomId]);

  useEffect(() => {
    // Leak C
    const interval = setInterval(() => {
      fetchMetrics(userId).then(setMetrics);
    }, 5000);
  }, [userId]);

  useEffect(() => {
    // Leak D
    fetch(`/api/rooms/${roomId}/users`)
      .then(r => r.json())
      .then(setOnlineUsers);
  }, [roomId]);

  useEffect(() => {
    // Leak E
    const report = generateHeavyReport(messages);
    reportCache[roomId] = report; // no eviction
  }, [messages, roomId]);

  return (/* ... */);
}
```

<details>
<summary>Solution</summary>

```jsx
// A module-level bounded cache to replace the unbounded one
const reportCache = new LRUCache(50); // max 50 rooms cached

function RealTimeDashboard({ roomId, userId }) {
  const [messages, setMessages]       = useState([]);
  const [onlineUsers, setOnlineUsers] = useState([]);
  const [metrics, setMetrics]         = useState({});

  // FIX A: Close WebSocket on cleanup
  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/room/${roomId}`);
    ws.onmessage = (e) => setMessages(prev => [...prev, JSON.parse(e.data)]);
    return () => {
      ws.onmessage = null;
      ws.close(); // ✅ close connection on unmount or roomId change
    };
  }, [roomId]);

  // FIX B: Remove event listener on cleanup
  // Also: use useCallback or store the handler to remove the exact same function
  useEffect(() => {
    function handleFocus() {
      fetchLatestMessages(roomId).then(setMessages);
    }
    window.addEventListener('focus', handleFocus);
    return () => window.removeEventListener('focus', handleFocus); // ✅
  }, [roomId]); // re-registers when roomId changes (correct)

  // FIX C: Clear interval on cleanup
  useEffect(() => {
    const interval = setInterval(() => {
      fetchMetrics(userId).then(setMetrics);
    }, 5000);
    return () => clearInterval(interval); // ✅
  }, [userId]);

  // FIX D: Cancel fetch on cleanup
  useEffect(() => {
    const controller = new AbortController();
    fetch(`/api/rooms/${roomId}/users`, { signal: controller.signal })
      .then(r => r.json())
      .then(setOnlineUsers)
      .catch(err => { if (err.name !== 'AbortError') console.error(err); });
    return () => controller.abort(); // ✅
  }, [roomId]);

  // FIX E: Use bounded LRU cache instead of unbounded object
  useEffect(() => {
    const report = generateHeavyReport(messages);
    reportCache.set(roomId, report); // ✅ LRU evicts old entries automatically
  }, [messages, roomId]);

  return (/* ... */);
}

// LEAKS FIXED:
// A: WebSocket stays open after unmount → ws.close() in cleanup
// B: Window event listener never removed → removeEventListener in cleanup
// C: setInterval fires after unmount → clearInterval in cleanup
// D: Fetch completes after unmount, setState called → AbortController in cleanup
// E: Unbounded cache grows without limit → LRU cache with max size
```

</details>

---

## 🔗 Related Topics

- [`anti-patterns/04-stale-closures.md`](./04-stale-closures.md) — Closures that cause both leaks and staleness
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md) — Hooks with proper cleanup
- [`javascript-core/08-memory-management.md`](../javascript-core/08-memory-management.md) — JS memory model and GC

---

<div align="center">

**`anti-patterns/` section complete!** 🎉

All 5 anti-patterns files done:
`01-prop-drilling.md` · `02-god-components.md` · `03-premature-optimization.md` · `04-stale-closures.md` · **`05-memory-leaks.md`** ✓

**Next section:** [`debugging/`](../debugging/) →

</div>
