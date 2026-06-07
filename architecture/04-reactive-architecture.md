# 04 — Reactive Architecture

> **"Reactive architecture is the acknowledgment that modern UIs are fundamentally about responding to change — user input, server data, time, other users. Instead of hiding that fundamental nature behind imperative control flow, reactive architecture embraces it. You describe what things should look like given current state, and the system figures out how to get there."**

Reactive architecture models applications as streams of state changes flowing through the system. Instead of imperatively commanding updates ("set this value to X"), you declaratively describe how the system should respond to inputs ("when this input changes, the output should be Y"). This mindset underlies React's component model, Redux's unidirectional flow, RxJS's observable streams, and the entire functional reactive programming (FRP) movement. This document covers the reactive model from first principles through practical patterns.

---

## 📚 Table of Contents

1. [The Reactive Model](#1-the-reactive-model)
2. [Reactive vs Imperative — The Fundamental Difference](#2-reactive-vs-imperative--the-fundamental-difference)
3. [Unidirectional Data Flow](#3-unidirectional-data-flow)
4. [The Observer Pattern at Scale](#4-the-observer-pattern-at-scale)
5. [Reactive State with Signals](#5-reactive-state-with-signals)
6. [RxJS — Reactive Streams](#6-rxjs--reactive-streams)
7. [Reactive Architecture Patterns](#7-reactive-architecture-patterns)
8. [Reactive Data Fetching](#8-reactive-data-fetching)
9. [Reactive Forms](#9-reactive-forms)
10. [Backpressure and Rate Limiting](#10-backpressure-and-rate-limiting)
11. [Reactive Architecture in React](#11-reactive-architecture-in-react)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Reactive Model

The reactive model treats everything as a stream of values over time:

```
IMPERATIVE MODEL:
  "At this moment, set the user name to Alice."
  State is manipulated through direct assignments.
  To know current value: read it when needed.
  Dependencies must be manually tracked.

REACTIVE MODEL:
  "user name = stream of values over time, currently Alice"
  State is derived from its sources automatically.
  When a source changes, all derived state updates automatically.
  Dependencies are declared once — the system manages updates.
```

```
Reactive streams:

time ──────────────────────────────────────────────────────►

searchQuery$:  ──"a"──"ap"──"app"──"appl"──"apple"──────────
products$:     ─────────────────────────────[A,B,C]──────────  (debounced fetch)
filteredCount$: ────────────────────────────[3]──────────────  (derived)
displayText$:  ────────────────────────────"3 results"───────  (derived from derived)

When searchQuery$ changes → products$ refetches → filteredCount$ updates → displayText$ updates
All automatically. No imperative "when search changes, call update, then re-render."
```

### The Four Properties of Reactive Systems (Reactive Manifesto)

```
1. RESPONSIVE:
   The system responds in a timely manner.
   Reactive architectures process events as they arrive — no blocking.

2. RESILIENT:
   The system remains responsive in the face of failure.
   Failures are isolated; the system recovers without user disruption.

3. ELASTIC:
   The system stays responsive under varying workload.
   Resources scale to match demand.

4. MESSAGE-DRIVEN:
   Components communicate via asynchronous message passing.
   Loose coupling — producers and consumers are independent.
```

---

## 2. Reactive vs Imperative — The Fundamental Difference

### Imperative Data Binding (the old way)

```javascript
// Imperative: manually update every dependent
let userName = "Alice";
let userStatus = "Active";

function updateUserDisplay() {
  // Must remember to call this whenever userName or userStatus changes
  document.getElementById("header").textContent = `${userName} (${userStatus})`;
  document.getElementById("greeting").textContent = `Hello, ${userName}!`;
  document.getElementById("profile-badge").textContent = userName[0];
}

// Every change requires manual coordination:
userName = "Bob";
updateUserDisplay(); // can't forget this!
updateProfileSidebar(); // and this!
updateRecentActivity(); // and this!
// Miss any one of them → stale UI
```

### Reactive Data Binding (the reactive way)

```typescript
// Reactive: declare relationships, system manages updates
// Angular-style reactive with signals:
const userName$ = signal("Alice");
const userStatus$ = signal("Active");

// Derived signals: automatically update when inputs change
const headerText$ = computed(() => `${userName$()} (${userStatus$()})`);
const greeting$ = computed(() => `Hello, ${userName$()}!`);
const profileBadge$ = computed(() => userName$()[0]);

// UI bindings: automatically re-render when signals change
// Template: {{ headerText$() }} — always current, never stale

// Update: just change the source, everything derived updates automatically
userName.set("Bob");
// headerText$, greeting$, profileBadge$ all update automatically
// No manual coordination needed
```

---

## 3. Unidirectional Data Flow

Unidirectional data flow is the most widely adopted reactive pattern in frontend development. Popularized by Flux/Redux, it enforces a single direction for data changes.

```
UNIDIRECTIONAL FLOW:

  ┌──────────────────────────────────────────────────────┐
  │                                                        │
  │   USER ACTION                                          │
  │   (click, type, scroll)                               │
  │        │                                              │
  │        ▼                                              │
  │   ACTION / EVENT                                      │
  │   { type: 'ADD_TO_CART', payload: { productId } }    │
  │        │                                              │
  │        ▼                                              │
  │   REDUCER / STATE MACHINE                            │
  │   (pure function: state + action → new state)        │
  │        │                                              │
  │        ▼                                              │
  │   STORE (single source of truth)                     │
  │   { cart: [...], user: {...} }                       │
  │        │                                              │
  │        ▼                                              │
  │   VIEW (derived from state)                          │
  │   React re-renders based on new state               │
  │        │                                              │
  │        └──────────────────────────────────────────── ┘
  │                                                        │
  │   (repeat — only flows one direction)                 │
  └──────────────────────────────────────────────────────┘
```

### Why Unidirectional?

```
BIDIRECTIONAL (two-way binding) problems:
  Component A changes state → Component B updates → which changes state →
  Component A updates → infinite loop or unpredictable behavior

  Hard to debug: "why did this value change?" — could be from anywhere

UNIDIRECTIONAL advantages:
  State can only change through explicit actions
  Every state change is traceable to a specific action
  Debugging: replay the action log to reproduce any state
  Testing: pure reducer functions are trivial to test
  DevTools: time-travel debugging, action replay
```

### Reducer Pattern

```typescript
// State shape
interface CartState {
  items: CartItem[];
  promoCode: string | null;
  isLoading: boolean;
  error: string | null;
}

// Actions (tagged union)
type CartAction =
  | { type: "ADD_ITEM"; payload: CartItem }
  | { type: "REMOVE_ITEM"; payload: { id: string } }
  | { type: "UPDATE_QTY"; payload: { id: string; qty: number } }
  | { type: "APPLY_PROMO"; payload: { code: string } }
  | { type: "CHECKOUT_START" }
  | { type: "CHECKOUT_SUCCESS" }
  | { type: "CHECKOUT_FAIL"; payload: { error: string } }
  | { type: "CLEAR" };

// Pure reducer (same input always produces same output)
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM": {
      const existing = state.items.find((i) => i.id === action.payload.id);
      return {
        ...state,
        items: existing
          ? state.items.map((i) =>
              i.id === action.payload.id
                ? { ...i, quantity: i.quantity + action.payload.quantity }
                : i,
            )
          : [...state.items, action.payload],
      };
    }

    case "REMOVE_ITEM":
      return {
        ...state,
        items: state.items.filter((i) => i.id !== action.payload.id),
      };

    case "UPDATE_QTY":
      return {
        ...state,
        items: state.items
          .map((i) =>
            i.id === action.payload.id
              ? { ...i, quantity: action.payload.qty }
              : i,
          )
          .filter((i) => i.quantity > 0), // remove if qty reaches 0
      };

    case "APPLY_PROMO":
      return { ...state, promoCode: action.payload.code };

    case "CHECKOUT_START":
      return { ...state, isLoading: true, error: null };

    case "CHECKOUT_SUCCESS":
      return { ...state, isLoading: false, items: [], promoCode: null };

    case "CHECKOUT_FAIL":
      return { ...state, isLoading: false, error: action.payload.error };

    case "CLEAR":
      return { items: [], promoCode: null, isLoading: false, error: null };

    default:
      return state;
  }
}
```

---

## 4. The Observer Pattern at Scale

At the heart of reactive architecture is the Observer pattern — subjects emit values, observers react.

### Observable Store (Simple)

```typescript
class ReactiveStore<T> {
  #state: T;
  #listeners = new Set<(state: T) => void>();

  constructor(initialState: T) {
    this.#state = initialState;
  }

  getState(): T {
    return this.#state;
  }

  setState(updater: T | ((prev: T) => T)): void {
    const next =
      typeof updater === "function"
        ? (updater as (prev: T) => T)(this.#state)
        : updater;

    if (Object.is(next, this.#state)) return; // skip if unchanged

    this.#state = next;
    this.#listeners.forEach((fn) => fn(next));
  }

  subscribe(listener: (state: T) => void): () => void {
    this.#listeners.add(listener);
    listener(this.#state); // immediately deliver current state

    return () => this.#listeners.delete(listener);
  }

  // Derived store — automatically recomputes when source changes
  derive<U>(selector: (state: T) => U): DerivedStore<U> {
    return new DerivedStore(this, selector);
  }
}

class DerivedStore<T> {
  #value: T;
  #listeners = new Set<(value: T) => void>();

  constructor(source: ReactiveStore<any>, selector: (s: any) => T) {
    this.#value = selector(source.getState());

    source.subscribe((state) => {
      const next = selector(state);
      if (Object.is(next, this.#value)) return; // skip if unchanged

      this.#value = next;
      this.#listeners.forEach((fn) => fn(next));
    });
  }

  get(): T {
    return this.#value;
  }

  subscribe(listener: (value: T) => void): () => void {
    this.#listeners.add(listener);
    listener(this.#value);
    return () => this.#listeners.delete(listener);
  }
}

// Usage
const cartStore = new ReactiveStore<CartState>({ items: [], isLoading: false });

// Derived stores update automatically
const itemCount$ = cartStore.derive((state) =>
  state.items.reduce((n, i) => n + i.quantity, 0),
);
const cartTotal$ = cartStore.derive((state) =>
  state.items.reduce((sum, i) => sum + i.price * i.quantity, 0),
);
const hasItems$ = cartStore.derive((state) => state.items.length > 0);

// Subscriptions
itemCount$.subscribe((count) => {
  document.getElementById("cart-badge")!.textContent = String(count);
});
```

---

## 5. Reactive State with Signals

Signals are the most refined form of reactive state — fine-grained reactivity where only the components that actually use a specific piece of state re-render when it changes.

### Implementing Signals from Scratch

```typescript
// Minimal signal implementation (inspired by SolidJS/Angular Signals)

let currentEffect: (() => void) | null = null;

function createSignal<T>(
  initialValue: T,
): [() => T, (v: T | ((prev: T) => T)) => void] {
  let value = initialValue;
  const listeners = new Set<() => void>();

  // Read: tracks which effects depend on this signal
  const read = (): T => {
    if (currentEffect) listeners.add(currentEffect);
    return value;
  };

  // Write: notifies all dependent effects
  const write = (nextOrUpdater: T | ((prev: T) => T)) => {
    const next =
      typeof nextOrUpdater === "function"
        ? (nextOrUpdater as (prev: T) => T)(value)
        : nextOrUpdater;

    if (Object.is(next, value)) return;
    value = next;
    listeners.forEach((fn) => fn());
  };

  return [read, write];
}

// Computed: automatically tracks signal dependencies
function createComputed<T>(fn: () => T): () => T {
  let cachedValue: T;
  let dirty = true;
  const listeners = new Set<() => void>();

  const recompute = () => {
    dirty = true;
    listeners.forEach((fn) => fn());
  };

  return (): T => {
    if (currentEffect) listeners.add(currentEffect);

    if (dirty) {
      const prev = currentEffect;
      currentEffect = recompute;
      cachedValue = fn(); // fn reads signals — they register recompute
      currentEffect = prev;
      dirty = false;
    }

    return cachedValue;
  };
}

// Effect: runs side effects when dependencies change
function createEffect(fn: () => void): () => void {
  const run = () => {
    const prev = currentEffect;
    currentEffect = run;
    fn(); // reads signals — they register `run` as a listener
    currentEffect = prev;
  };

  run(); // run immediately to track initial dependencies
  return () => {
    /* cleanup: would need WeakRef tracking */
  };
}

// Usage:
const [count, setCount] = createSignal(0);
const [name, setName] = createSignal("Alice");
const doubled = createComputed(() => count() * 2);
const greeting = createComputed(() => `Hello, ${name()}! Count: ${doubled()}`);

createEffect(() => {
  document.getElementById("output")!.textContent = greeting();
});

// Updates cascade automatically:
setCount(5); // output: "Hello, Alice! Count: 10"
setName("Bob"); // output: "Hello, Bob! Count: 10"
```

---

## 6. RxJS — Reactive Streams

RxJS models everything as Observable streams — sequences of values over time that can be transformed, combined, and controlled.

### Core Observable Operations

```typescript
import {
  Observable,
  Subject,
  BehaviorSubject,
  combineLatest,
  merge,
  fromEvent,
} from "rxjs";
import {
  map,
  filter,
  debounceTime,
  switchMap,
  distinctUntilChanged,
  catchError,
  takeUntil,
  shareReplay,
  startWith,
  scan,
  throttleTime,
  mergeMap,
  concatMap,
  exhaustMap,
} from "rxjs/operators";

// ─── Creating Observables ────────────────────────────────────

// From DOM events
const clicks$ = fromEvent(button, "click");
const keydowns$ = fromEvent<KeyboardEvent>(input, "keydown");
const scroll$ = fromEvent(window, "scroll");

// Subject: both Observable and Observer (can push values)
const action$ = new Subject<CartAction>();

// BehaviorSubject: always has a current value
const cart$ = new BehaviorSubject<CartState>(initialState);

// ─── Transforming ────────────────────────────────────────────

const searchQuery$ = fromEvent<InputEvent>(searchInput, "input").pipe(
  map((e) => (e.target as HTMLInputElement).value), // extract value
  filter((q) => q.length >= 2), // ignore short queries
  debounceTime(300), // wait for pause in typing
  distinctUntilChanged(), // skip if same value
);

// ─── Higher-Order Operators (handle inner Observables) ───────

// switchMap: cancel previous inner Observable when new value arrives
// Best for: search (cancel old request when user keeps typing)
const results$ = searchQuery$.pipe(
  switchMap((query) => searchApi.query(query)), // inner Observable
  catchError(() => EMPTY), // handle errors gracefully
);

// concatMap: queue inner Observables, execute in order
// Best for: sequential operations, save drafts one at a time
const savedDraft$ = saveClicks$.pipe(
  concatMap((draft) => draftApi.save(draft)),
);

// mergeMap: execute all inner Observables concurrently
// Best for: independent parallel requests
const productDetails$ = productIds$.pipe(
  mergeMap((id) => productApi.get(id)), // all fire at once
);

// exhaustMap: ignore new values while inner Observable is active
// Best for: prevent double-submits on form submit button
const submitResult$ = formSubmit$.pipe(
  exhaustMap((data) => api.submit(data)), // ignores clicks while request is pending
);

// ─── Combining Observables ───────────────────────────────────

// combineLatest: emit when ANY input emits (with latest of all)
const cartTotal$ = combineLatest([cartItems$, prices$]).pipe(
  map(([items, prices]) =>
    items.reduce(
      (sum, item) => sum + (prices[item.id] ?? item.price) * item.quantity,
      0,
    ),
  ),
);

// merge: forward emissions from any source
const anyCartChange$ = merge(itemAdded$, itemRemoved$, quantityChanged$);
```

### Real-World RxJS: Live Search

```typescript
function LiveSearch({ onResults }: { onResults: (r: Product[]) => void }) {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    const input = inputRef.current!;
    const destroy$ = new Subject<void>();

    fromEvent<InputEvent>(input, 'input').pipe(
      map(e => (e.target as HTMLInputElement).value.trim()),
      debounceTime(300),
      distinctUntilChanged(),
      filter(query => query.length >= 2),
      switchMap(query =>
        from(searchApi.products(query)).pipe(
          catchError(() => of([])) // empty array on error
        )
      ),
      takeUntil(destroy$), // unsubscribe when component unmounts
    )
    .subscribe(results => onResults(results));

    return () => {
      destroy$.next();
      destroy$.complete();
    };
  }, [onResults]);

  return <input ref={inputRef} placeholder="Search products..." />;
}
```

---

## 7. Reactive Architecture Patterns

### FLUX Pattern (Unidirectional Redux-style)

```typescript
// Action creators
const cartActions = {
  addItem: (item: CartItem) => ({ type: "ADD_ITEM" as const, payload: item }),
  removeItem: (id: string) => ({
    type: "REMOVE_ITEM" as const,
    payload: { id },
  }),
  checkout: () => ({ type: "CHECKOUT" as const }),
};

// Redux slice equivalent with Zustand
const useCartStore = create<CartStore>()((set, get) => ({
  items: [],
  status: "idle" as "idle" | "checking-out" | "success" | "error",

  // Actions always go through the store
  addItem: (item) => set((s) => ({ items: addOrUpdateItem(s.items, item) })),
  removeItem: (id) =>
    set((s) => ({ items: s.items.filter((i) => i.id !== id) })),
  checkout: async () => {
    set({ status: "checking-out" });
    try {
      await checkoutApi.place(get().items);
      set({ status: "success", items: [] });
    } catch {
      set({ status: "error" });
    }
  },
}));
```

### MVI Pattern (Model-View-Intent)

```typescript
// MVI: Intent → Model → View

// INTENT: translate user actions into intentions
function createIntents(actions$: Observable<Action>) {
  return {
    addToCart$:     actions$.pipe(filter(a => a.type === 'ADD_ITEM'),     map(a => a.payload)),
    removeFromCart$: actions$.pipe(filter(a => a.type === 'REMOVE_ITEM'), map(a => a.payload.id)),
    checkout$:      actions$.pipe(filter(a => a.type === 'CHECKOUT')),
  };
}

// MODEL: derive state from intents
function createModel(intents: ReturnType<typeof createIntents>) {
  return merge(
    intents.addToCart$.pipe(map(item => ({ type: 'add', item } as const))),
    intents.removeFromCart$.pipe(map(id => ({ type: 'remove', id } as const))),
  ).pipe(
    scan((cart, event) => {
      if (event.type === 'add') return addOrUpdateItem(cart, event.item);
      if (event.type === 'remove') return cart.filter(i => i.id !== event.id);
      return cart;
    }, [] as CartItem[]),
    startWith([] as CartItem[]),
    shareReplay(1),
  );
}

// VIEW: subscribe to model, render
function CartView() {
  const [cart, setCart] = useState<CartItem[]>([]);
  const actions$ = useMemo(() => new Subject<Action>(), []);
  const dispatch  = useCallback((action: Action) => actions$.next(action), [actions$]);

  useEffect(() => {
    const intents = createIntents(actions$);
    const cart$   = createModel(intents);
    const sub     = cart$.subscribe(setCart);
    return () => sub.unsubscribe();
  }, [actions$]);

  return (
    <div>
      {cart.map(item => (
        <CartItemRow
          key={item.id}
          item={item}
          onRemove={() => dispatch({ type: 'REMOVE_ITEM', payload: { id: item.id } })}
        />
      ))}
    </div>
  );
}
```

---

## 8. Reactive Data Fetching

TanStack Query implements reactive data fetching — queries are reactive: they refetch on window focus, network reconnect, cache invalidation.

```typescript
// Reactive query: automatically refetches when conditions change
function useProductSearch(query: string, filters: ProductFilters) {
  return useQuery({
    queryKey: ["products", query, filters], // key change → refetch
    queryFn: () => productsApi.search({ query, ...filters }),
    enabled: query.length >= 2, // reactive: only runs when enabled
    staleTime: 30_000, // fresh for 30 seconds
    refetchOnWindowFocus: true, // reactive: refetch when tab focused
    refetchInterval: 60_000, // reactive: refetch every minute
  });
}

// Reactive dependent queries
function useOrderWithUser(orderId: string) {
  const orderQuery = useQuery({
    queryKey: ["order", orderId],
    queryFn: () => ordersApi.get(orderId),
  });

  const userQuery = useQuery({
    queryKey: ["user", orderQuery.data?.customerId],
    queryFn: () => usersApi.get(orderQuery.data!.customerId),
    enabled: !!orderQuery.data?.customerId, // reactive: only when orderId resolves
  });

  return { order: orderQuery.data, user: userQuery.data };
}
```

---

## 9. Reactive Forms

Reactive forms model form state as a reactive graph where field values, validation state, and derived states all update automatically.

```typescript
// Custom reactive form implementation
class ReactiveFormControl<T> {
  readonly value$: BehaviorSubject<T>;
  readonly touched$: BehaviorSubject<boolean>;
  readonly dirty$: BehaviorSubject<boolean>;
  readonly errors$: Observable<string[]>;
  readonly valid$: Observable<boolean>;

  constructor(
    initialValue: T,
    validators: ((value: T) => string | null)[] = [],
  ) {
    this.value$ = new BehaviorSubject(initialValue);
    this.touched$ = new BehaviorSubject(false);
    this.dirty$ = new BehaviorSubject(false);

    this.errors$ = this.value$.pipe(
      map((value) =>
        validators.map((v) => v(value)).filter((e): e is string => e !== null),
      ),
    );

    this.valid$ = this.errors$.pipe(map((errors) => errors.length === 0));
  }

  setValue(value: T): void {
    this.value$.next(value);
    this.dirty$.next(true);
  }

  markAsTouched(): void {
    this.touched$.next(true);
  }

  reset(value: T): void {
    this.value$.next(value);
    this.touched$.next(false);
    this.dirty$.next(false);
  }
}

// Reactive form group
class ReactiveFormGroup {
  readonly controls: Record<string, ReactiveFormControl<any>>;
  readonly value$: Observable<Record<string, any>>;
  readonly valid$: Observable<boolean>;

  constructor(controls: Record<string, ReactiveFormControl<any>>) {
    this.controls = controls;

    this.value$ = combineLatest(
      Object.fromEntries(
        Object.entries(controls).map(([key, ctrl]) => [key, ctrl.value$]),
      ),
    );

    this.valid$ = combineLatest(
      Object.values(controls).map((ctrl) => ctrl.valid$),
    ).pipe(map((valids) => valids.every(Boolean)));
  }
}

// Usage
const loginForm = new ReactiveFormGroup({
  email: new ReactiveFormControl("", [
    (v) => (!v ? "Email is required" : null),
    (v) => (!v.includes("@") ? "Invalid email" : null),
  ]),
  password: new ReactiveFormControl("", [
    (v) => (!v ? "Password is required" : null),
    (v) => (v.length < 8 ? "Minimum 8 characters" : null),
  ]),
});

// Subscribe to form validity for submit button
loginForm.valid$.subscribe((isValid) => {
  submitButton.disabled = !isValid;
});
```

---

## 10. Backpressure and Rate Limiting

When the producer emits faster than the consumer can handle, reactive systems need backpressure strategies.

```typescript
import {
  throttleTime,
  auditTime,
  sampleTime,
  bufferTime,
} from "rxjs/operators";

// THROTTLE: emit at most once per interval (leading edge)
const throttledScroll$ = scroll$.pipe(
  throttleTime(100), // max 10 events/second
);

// AUDIT: emit the latest value at each interval (trailing edge)
const auditedScroll$ = scroll$.pipe(
  auditTime(100), // emit latest value 100ms after last emission
);

// BUFFER: collect events, emit as array
const bufferedClicks$ = clicks$.pipe(
  bufferTime(1000), // emit array of all clicks per second
  filter((clicks) => clicks.length > 0),
);

// DEBOUNCE: emit after silence period
const debouncedSearch$ = searchInput$.pipe(
  debounceTime(300), // emit 300ms after user stops typing
);

// Real-time price updates: only render what we can
const displayPrices$ = priceUpdates$.pipe(
  throttleTime(1000 / 60, asyncScheduler, { leading: true, trailing: true }),
  // max 60 updates/second (one per frame)
);
```

---

## 11. Reactive Architecture in React

### React as a Reactive System

React is itself a reactive system: state changes cause a reactive re-render of the component tree.

```typescript
// React's reactive model:
// state = input signal
// JSX   = computed output from state
// useEffect = side effect triggered by state changes

function CartBadge() {
  // count is reactive — component re-renders when it changes
  const count = useCartStore(s => s.items.reduce((n, i) => n + i.quantity, 0));

  return <span className="badge">{count}</span>;
}
// When cart changes → count changes → CartBadge re-renders automatically
```

### useSyncExternalStore for Reactive External Stores

```typescript
// Connect any reactive store to React's rendering cycle
function useReactiveStore<T>(store: ReactiveStore<T>): T {
  return useSyncExternalStore(
    (callback) => store.subscribe(callback),  // subscribe
    ()         => store.getState(),            // snapshot
    ()         => store.getState(),            // server snapshot
  );
}

// Usage
function CartCount() {
  const count = useReactiveStore(itemCountStore);
  return <span>{count}</span>;
}
```

### useReducer for Complex Reactive State

```typescript
// useReducer is React's built-in unidirectional state pattern
function Cart() {
  const [state, dispatch] = useReducer(cartReducer, initialCartState);

  // All state changes go through dispatch (unidirectional)
  const handleAdd    = (item: CartItem) => dispatch({ type: 'ADD_ITEM', payload: item });
  const handleRemove = (id: string)     => dispatch({ type: 'REMOVE_ITEM', payload: { id } });

  // Derived state: computed from current state
  const total    = state.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  const hasItems = state.items.length > 0;

  return (
    <div>
      {state.items.map(item => (
        <CartItem key={item.id} item={item} onRemove={() => handleRemove(item.id)} />
      ))}
      <p>Total: ${total.toFixed(2)}</p>
      {hasItems && <button onClick={() => dispatch({ type: 'CHECKOUT' })}>Checkout</button>}
    </div>
  );
}
```

---

## 12. Good Practices

### ✅ Keep effects at the edges

```typescript
// ✅ Pure reactive core (no side effects in the data flow)
const cartTotal$ = cartItems$.pipe(
  map((items) => items.reduce((sum, i) => sum + i.price * i.quantity, 0)),
);

// Side effects at the "edge" only (subscriptions)
cartTotal$.subscribe((total) => {
  analytics.track("cart_total_changed", { total }); // side effect here
  updateDOM("cart-total", formatCurrency(total)); // side effect here
});
```

### ✅ Always unsubscribe

```typescript
// ✅ React: unsubscribe in useEffect cleanup
useEffect(() => {
  const subscription = someObservable$.subscribe(handleUpdate);
  return () => subscription.unsubscribe(); // cleanup
}, []);

// ✅ RxJS: use takeUntil with a destroy subject
const destroy$ = new Subject<void>();

someObservable$.pipe(takeUntil(destroy$)).subscribe(handleUpdate);

// On component destroy:
destroy$.next();
destroy$.complete();
```

### ✅ Prefer declarative over imperative composition

```typescript
// ✅ Declarative: describe what you want
const filteredProducts$ = combineLatest([allProducts$, activeFilters$]).pipe(
  map(([products, filters]) => applyFilters(products, filters)),
);

// ❌ Imperative: manually coordinate updates
allProducts$.subscribe((products) => {
  const filters = activeFilters$.getValue();
  const filtered = applyFilters(products, filters);
  filteredProductsSubject.next(filtered);
});
activeFilters$.subscribe((filters) => {
  const products = allProducts$.getValue();
  const filtered = applyFilters(products, filters);
  filteredProductsSubject.next(filtered);
});
```

---

## 13. Bad Practices

### ❌ Nested subscriptions (subscribe inside subscribe)

```typescript
// ❌ Nested: creates memory leaks and race conditions
userId$.subscribe((userId) => {
  userApi.getUser(userId).subscribe((user) => {
    // ← nested!
    render(user);
  });
  // Previous inner subscription is never cancelled when userId changes!
});

// ✅ switchMap handles this correctly
userId$
  .pipe(
    switchMap((userId) => from(userApi.getUser(userId))), // cancels previous
  )
  .subscribe((user) => render(user));
```

### ❌ Hot observables without replay for late subscribers

```typescript
// ❌ Subject without replay: late subscribers miss values
const user$ = new Subject<User>();
user$.next(currentUser); // emitted before anything subscribes

// Later: subscribe doesn't receive the value
user$.subscribe((user) => console.log(user)); // receives nothing

// ✅ BehaviorSubject always has a current value
const user$ = new BehaviorSubject<User | null>(null);
user$.next(currentUser);

user$.subscribe((user) => console.log(user)); // immediately receives currentUser
```

### ❌ Mutating state instead of producing new state

```typescript
// ❌ Mutating state in reducer/reactive update
function cartReducer(state: CartState, action: CartAction): CartState {
  if (action.type === "ADD_ITEM") {
    state.items.push(action.payload); // mutates! React won't detect change
    return state;
  }
  return state;
}

// ✅ Always return new state object
function cartReducer(state: CartState, action: CartAction): CartState {
  if (action.type === "ADD_ITEM") {
    return { ...state, items: [...state.items, action.payload] };
  }
  return state;
}
```

---

## 14. Common Mistakes

### Mistake 1 — Not handling errors in streams

```typescript
// ❌ Unhandled error terminates the stream
data$
  .pipe(
    switchMap((id) => from(api.fetch(id))),
    // Error in api.fetch → stream terminates → no more updates
  )
  .subscribe(render);

// ✅ Catch and recover within the inner observable
data$
  .pipe(
    switchMap((id) =>
      from(api.fetch(id)).pipe(
        catchError((err) => {
          console.error(err);
          return of(null); // recover with null — outer stream continues
        }),
      ),
    ),
  )
  .subscribe((data) => {
    if (data) render(data);
    else renderError();
  });
```

### Mistake 2 — Memory leaks from untracked subscriptions

```typescript
// ❌ Subscription created but never cleaned up
class MyService {
  init() {
    dataStream$.subscribe(this.handleData); // never unsubscribed!
  }
  // When service is destroyed: subscription persists → memory leak
}

// ✅ Track and cleanup
class MyService {
  #subscription: Subscription | null = null;

  init() {
    this.#subscription = dataStream$.subscribe(this.handleData);
  }

  destroy() {
    this.#subscription?.unsubscribe();
    this.#subscription = null;
  }
}
```

### Mistake 3 — Using BehaviorSubject as a two-way binding mechanism

```typescript
// ❌ Treating BehaviorSubject as mutable variable (loses reactivity benefits)
const name$ = new BehaviorSubject("Alice");

// Directly subscribing AND writing → bypasses the reactive model
const currentName = name$.getValue(); // reads current value
name$.next(newName); // writes without going through reducer

// ✅ For complex state: use a store with explicit mutations
const nameStore = create((set) => ({
  name: "Alice",
  setName: (name: string) => set({ name }),
}));
```

---

## 15. Interview-Level Explanation

> **"What is reactive architecture? How does unidirectional data flow work?"**

**Strong answer:**

> "Reactive architecture models applications as systems that respond to streams of input — user actions, server data, time — rather than explicitly commanding each state update. Instead of saying 'when X happens, do Y, then do Z,' you declare: 'this value is a function of these inputs' and the system handles propagation automatically.
>
> The most concrete reactive pattern in frontend is unidirectional data flow, popularized by Redux and Flux. It has three moving parts: actions describe what happened in the application (user added item to cart), reducers are pure functions that take current state and an action and return the next state without mutation, and the store holds the single source of truth and notifies subscribers when state changes. Components derive their view from the store and dispatch actions in response to user input. The cycle is strictly one-directional: action → reducer → store → view → action. This makes state changes predictable and traceable — any state can be reproduced by replaying the action log.
>
> React itself is reactive: state is the signal, and the component tree is a derived computation from that signal. When state changes, React recomputes what the view should look like and applies the minimum necessary DOM updates. The virtual DOM diff is React's mechanism for doing that efficiently.
>
> Signals take this further — fine-grained reactivity where only the specific component that reads a specific signal re-renders when that signal changes. Angular 17+, SolidJS, and Vue 3's reactivity system all use this model. The difference from React: React re-renders the component subtree from the changed state down; signals track exactly which computations depend on which pieces of state and only recompute those.
>
> For async streams — search, WebSocket data, polling — RxJS provides a rich composition toolkit. The higher-order operators are the key piece: `switchMap` cancels a previous in-flight request when a new value arrives (perfect for live search), `exhaustMap` ignores new values while one is being processed (prevents double-submits), `concatMap` queues operations in order (sequential saves). These operators encode complex async coordination in a declarative, composable way that would require significant imperative boilerplate otherwise."

---

## 16. Exercises

### Exercise 1 — Build a reactive search store

Implement a reactive store for a product search that:

- Accepts search query and filter changes
- Debounces API calls by 300ms
- Cancels previous requests when new query arrives
- Tracks loading state
- Exposes derived count of results

```typescript
interface SearchState {
  query: string;
  filters: ProductFilters;
  results: Product[];
  loading: boolean;
  error: string | null;
}
```

<details>
<summary>Solution</summary>

```typescript
import { create } from 'zustand';
import { debounce } from 'lodash-es';

interface SearchStore extends SearchState {
  setQuery:   (query: string) => void;
  setFilters: (filters: ProductFilters) => void;
  resultCount: number; // derived
}

const useSearchStore = create<SearchStore>((set, get) => {
  // Debounced search — cancellable via AbortController
  let abortController: AbortController | null = null;

  const performSearch = debounce(async () => {
    const { query, filters } = get();
    if (query.length < 2) {
      set({ results: [], loading: false });
      return;
    }

    // Cancel previous request
    abortController?.abort();
    abortController = new AbortController();

    set({ loading: true, error: null });

    try {
      const results = await productsApi.search(
        { query, ...filters },
        { signal: abortController.signal }
      );
      set({ results, loading: false });
    } catch (err) {
      if (err.name === 'AbortError') return; // cancelled — ignore
      set({ loading: false, error: 'Search failed. Please try again.' });
    }
  }, 300);

  return {
    query:   '',
    filters: {},
    results: [],
    loading: false,
    error:   null,

    setQuery(query) {
      set({ query });
      performSearch();
    },

    setFilters(filters) {
      set({ filters });
      performSearch();
    },

    // Derived — computed inline via Zustand selector
    get resultCount() {
      return get().results.length;
    },
  };
});

// Usage
function SearchBar() {
  const { query, setQuery } = useSearchStore();
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}

function SearchResults() {
  const { results, loading, error, resultCount } = useSearchStore();
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return (
    <div>
      <p>{resultCount} results</p>
      {results.map(p => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md) — Async patterns underlying reactive systems
- [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md) — Observer pattern in depth
- [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md) — State architecture including reactive patterns
- [`system-design/06-event-driven-frontend.md`](../system-design/06-event-driven-frontend.md) — Event-driven patterns complement reactive
- [`architecture/01-layered-architecture.md`](./01-layered-architecture.md) — Where reactive state fits in layered architecture

---

<div align="center">

**`architecture/` section complete!** 🎉

All 4 architecture files done:
`01-layered-architecture.md` · `02-clean-architecture.md` · `03-domain-driven-design.md` · **`04-reactive-architecture.md`** ✓

**Next section:** [`rendering/`](../rendering/) →

</div>
