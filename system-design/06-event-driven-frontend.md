# 06 — Event-Driven Frontend

> **"Direct function calls create tight coupling. Event emission creates loose coupling. The difference is whether the caller knows about the callee. In a direct call, it must. In an event, it doesn't — and that ignorance is architectural freedom."**

Event-driven architecture on the frontend replaces direct module-to-module calls with a publish/subscribe model: modules emit events describing what happened, and other modules react by subscribing to the events they care about. This decouples features from each other, enables extensibility without modification, and scales from simple component communication to complex multi-team micro-frontend coordination. This document covers the full spectrum: from DOM custom events to typed event buses, command patterns, reactive streams, and the tradeoffs involved.

---

## 📚 Table of Contents

1. [Why Event-Driven Architecture](#1-why-event-driven-architecture)
2. [DOM Custom Events — The Native Foundation](#2-dom-custom-events--the-native-foundation)
3. [The Event Bus Pattern](#3-the-event-bus-pattern)
4. [Typed Event Systems in TypeScript](#4-typed-event-systems-in-typescript)
5. [Command vs Event — The Distinction](#5-command-vs-event--the-distinction)
6. [Event Sourcing on the Frontend](#6-event-sourcing-on-the-frontend)
7. [Reactive Streams with RxJS](#7-reactive-streams-with-rxjs)
8. [Events in React — Context, State, and Callbacks](#8-events-in-react--context-state-and-callbacks)
9. [Cross-Feature Communication with Events](#9-cross-feature-communication-with-events)
10. [WebSocket Events and Real-Time Integration](#10-websocket-events-and-real-time-integration)
11. [Event Logging and Analytics](#11-event-logging-and-analytics)
12. [Testing Event-Driven Code](#12-testing-event-driven-code)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. Why Event-Driven Architecture

### The Direct Call Problem

```typescript
// Module A directly calls Module B
// features/cart/cartService.ts
import { notificationService } from "@/features/notifications";
import { analyticsService } from "@/infrastructure/analytics";
import { orderService } from "@/features/orders";

async function checkout(cart: Cart) {
  const order = await cartApi.checkout(cart);

  // Direct calls: Cart now knows about Notifications, Analytics, Orders
  notificationService.show("Order placed!");
  analyticsService.track("checkout_completed", { orderId: order.id });
  orderService.refresh();

  return order;
}

// Problems:
// - Cart imports 3 other modules — tightly coupled
// - Adding a 4th reaction (email, loyalty points) = modify Cart
// - Testing Cart requires mocking Notifications + Analytics + Orders
// - Notifications can't exist without Cart module
```

### The Event-Driven Solution

```typescript
// features/cart/cartService.ts — event-driven
import { eventBus } from "@/infrastructure/eventBus";

async function checkout(cart: Cart) {
  const order = await cartApi.checkout(cart);

  // Cart only knows it needs to tell the world what happened
  eventBus.publish("checkout:completed", {
    orderId: order.id,
    total: order.total,
    items: cart.items,
    cartId: cart.id,
  });

  return order;
}

// Each reaction is registered in its own module:
// features/notifications/index.ts:
eventBus.subscribe("checkout:completed", ({ orderId }) =>
  notificationService.show(`Order #${orderId} placed!`),
);

// infrastructure/analytics/index.ts:
eventBus.subscribe("checkout:completed", (event) =>
  analytics.track("checkout_completed", event),
);

// features/orders/index.ts:
eventBus.subscribe("checkout:completed", () => ordersCache.invalidate());

// Adding loyalty points: just add a new subscriber
// No change to cartService.ts
```

---

## 2. DOM Custom Events — The Native Foundation

The browser's native event system supports custom events — zero dependencies, works across any framework.

### Creating and Dispatching

```typescript
// Dispatch a custom event
function addItemToCart(product: Product) {
  // ... add to cart logic ...

  // Dispatch custom event up the DOM tree
  const event = new CustomEvent("cart:item-added", {
    bubbles: true, // bubbles up to document
    composed: true, // crosses shadow DOM boundaries
    cancelable: true, // can be prevented
    detail: {
      // event data
      productId: product.id,
      productName: product.name,
      price: product.price,
      quantity: 1,
    },
  });

  document.dispatchEvent(event);
  // Or dispatch from specific element for scoped bubbling:
  cartButton.dispatchEvent(event);
}
```

### Listening to Custom Events

```typescript
// Listen globally
document.addEventListener("cart:item-added", (event: CustomEvent) => {
  updateCartBadge(event.detail);
});

// Listen scoped to a container
const container = document.getElementById("app");
container.addEventListener("cart:item-added", handleCartUpdate);

// One-time listener
document.addEventListener(
  "user:onboarded",
  () => {
    showWelcomeTour();
  },
  { once: true },
);

// With cleanup
const controller = new AbortController();
document.addEventListener("cart:item-added", handler, {
  signal: controller.signal,
});
// Remove later:
controller.abort();
```

### TypeScript Custom Event Types

```typescript
// Type-safe custom events
interface CartItemAddedDetail {
  productId: string;
  productName: string;
  price: number;
  quantity: number;
}

declare global {
  interface DocumentEventMap {
    "cart:item-added": CustomEvent<CartItemAddedDetail>;
    "auth:logged-out": CustomEvent<void>;
    "theme:changed": CustomEvent<{ theme: "light" | "dark" }>;
  }
}

// Now TypeScript knows the event detail type:
document.addEventListener("cart:item-added", (event) => {
  event.detail.productId; // ✅ typed
  event.detail.price; // ✅ typed
});
```

---

## 3. The Event Bus Pattern

For larger applications, an explicit event bus provides more control than DOM events.

```typescript
// infrastructure/eventBus/EventBus.ts

type Handler<T = unknown> = (data: T) => void;
type Unsubscribe = () => void;

interface EventBusOptions {
  maxListeners?: number; // warn if exceeded (helps catch leaks)
  debug?: boolean; // log all events
}

export class EventBus<
  TEventMap extends Record<string, unknown> = Record<string, unknown>,
> {
  #listeners = new Map<string, Set<Handler>>();
  #maxListeners: number;
  #debug: boolean;

  constructor(options: EventBusOptions = {}) {
    this.#maxListeners = options.maxListeners ?? 100;
    this.#debug = options.debug ?? false;
  }

  subscribe<K extends keyof TEventMap>(
    event: K,
    handler: Handler<TEventMap[K]>,
  ): Unsubscribe {
    const key = String(event);
    if (!this.#listeners.has(key)) {
      this.#listeners.set(key, new Set());
    }

    const listeners = this.#listeners.get(key)!;

    if (listeners.size >= this.#maxListeners) {
      console.warn(
        `EventBus: ${listeners.size} listeners for '${key}' — possible leak`,
      );
    }

    listeners.add(handler as Handler);

    if (this.#debug) {
      console.debug(
        `[EventBus] subscribed to '${key}' (${listeners.size} total)`,
      );
    }

    // Return unsubscribe function
    return () => {
      listeners.delete(handler as Handler);
      if (listeners.size === 0) this.#listeners.delete(key);
    };
  }

  publish<K extends keyof TEventMap>(event: K, data: TEventMap[K]): void {
    const key = String(event);
    const listeners = this.#listeners.get(key);

    if (this.#debug) {
      console.debug(`[EventBus] published '${key}':`, data);
    }

    if (!listeners || listeners.size === 0) return;

    // Copy to prevent mutation during iteration
    for (const handler of [...listeners]) {
      try {
        handler(data);
      } catch (err) {
        console.error(`[EventBus] error in '${key}' handler:`, err);
      }
    }
  }

  once<K extends keyof TEventMap>(
    event: K,
    handler: Handler<TEventMap[K]>,
  ): Unsubscribe {
    const unsubscribe = this.subscribe(event, (data) => {
      handler(data);
      unsubscribe();
    });
    return unsubscribe;
  }

  unsubscribeAll(event?: keyof TEventMap): void {
    if (event) {
      this.#listeners.delete(String(event));
    } else {
      this.#listeners.clear();
    }
  }

  listenerCount(event: keyof TEventMap): number {
    return this.#listeners.get(String(event))?.size ?? 0;
  }
}
```

---

## 4. Typed Event Systems in TypeScript

Centralizing event type definitions prevents naming collisions and provides autocomplete.

```typescript
// infrastructure/eventBus/events.ts — the contract

// Domain events
interface DomainEvents {
  // Auth
  "auth:login-success": {
    user: User;
    token: string;
    method: "password" | "oauth" | "magic-link";
  };
  "auth:logout": { reason: "user-initiated" | "session-expired" | "forced" };
  "auth:token-refreshed": { token: string; expiresAt: number };

  // Cart
  "cart:item-added": {
    productId: string;
    quantity: number;
    source: "plp" | "pdp" | "wishlist";
  };
  "cart:item-removed": { productId: string; reason: "manual" | "out-of-stock" };
  "cart:item-qty-changed": {
    productId: string;
    oldQty: number;
    newQty: number;
  };
  "cart:coupon-applied": { code: string; discount: number };
  "cart:checkout-started": { cartId: string; itemCount: number; total: number };

  // Orders
  "order:placed": { orderId: string; total: number; paymentMethod: string };
  "order:status-changed": {
    orderId: string;
    from: OrderStatus;
    to: OrderStatus;
  };
  "order:cancelled": { orderId: string; reason: string };

  // UI
  "ui:theme-changed": { theme: "light" | "dark" };
  "ui:language-changed": { locale: string };
  "ui:notification-shown": {
    id: string;
    type: "info" | "success" | "warning" | "error";
  };

  // Error
  "app:error": {
    message: string;
    severity: "low" | "medium" | "high";
    source: string;
  };
}

// Create typed event bus
export const eventBus = new EventBus<DomainEvents>({
  maxListeners: 50,
  debug: import.meta.env.DEV,
});

// TypeScript enforces correct event names and payload shapes:
eventBus.publish("cart:item-added", {
  productId: "42",
  quantity: 1,
  source: "pdp",
}); // ✅

eventBus.publish("cart:item-added", {
  productId: "42",
}); // ❌ TypeScript error: missing quantity and source

eventBus.publish("cart:unknown", {}); // ❌ TypeScript error: unknown event
```

---

## 5. Command vs Event — The Distinction

Events and commands are both messages, but with fundamentally different semantics:

```
EVENT:
  "Something happened" — past tense, declarative
  Publisher doesn't know or care what happens next
  Multiple subscribers can react
  Example: 'user:registered', 'order:placed', 'payment:failed'

COMMAND:
  "Please do this" — imperative, directed
  Sender expects a specific thing to happen
  Usually one handler
  Example: 'sendWelcomeEmail', 'invalidateCart', 'redirectToLogin'
```

### Events (Broad Cast)

```typescript
// Events: publisher doesn't dictate the reaction
eventBus.publish("user:registered", { userId, email, plan });

// Multiple independent reactions:
// - email service sends welcome email
// - analytics tracks signup
// - onboarding system creates initial data
// - billing service starts trial
// Publisher knows none of these details
```

### Commands (Targeted)

```typescript
// Commands: caller explicitly invokes a specific action
// Better modeled as direct function calls or service methods:

// Service method (command):
emailService.sendWelcomeEmail(user.email);
// Caller knows who handles it — it's a direct call
// When "do this" is the right model: use a function, not an event

// Events are for "this happened, anyone interested?"
// Commands are for "please do this specific thing"
```

### CQRS-Inspired Pattern

```typescript
// Separate reads and writes via events
class UserService {
  // COMMAND: performs the mutation
  async createUser(data: CreateUserDto): Promise<User> {
    const user = await usersApi.create(data);

    // EVENT: announces what happened (state change)
    eventBus.publish("user:created", { user });

    return user;
  }
}

// READ side: reacts to events, updates its own view
class UserListStore {
  #users: User[] = [];

  constructor() {
    eventBus.subscribe("user:created", ({ user }) => this.#users.push(user));
    eventBus.subscribe("user:updated", ({ user }) => this.#replaceUser(user));
    eventBus.subscribe("user:deleted", ({ userId }) =>
      this.#removeUser(userId),
    );
  }

  get users() {
    return this.#users;
  }
}
```

---

## 6. Event Sourcing on the Frontend

Event sourcing stores state as a sequence of events, deriving current state by replaying them.

```typescript
// All changes recorded as events
type AppEvent =
  | { type: "USER_LOGGED_IN"; payload: { user: User }; timestamp: number }
  | {
      type: "ITEM_ADDED_TO_CART";
      payload: { item: CartItem };
      timestamp: number;
    }
  | {
      type: "COUPON_APPLIED";
      payload: { code: string; discount: number };
      timestamp: number;
    }
  | {
      type: "CHECKOUT_STARTED";
      payload: { cartId: string };
      timestamp: number;
    };

class EventStore {
  #events: AppEvent[] = [];

  append(event: AppEvent) {
    this.#events.push({ ...event, timestamp: Date.now() });
  }

  // Derive current state by replaying events
  replay(): AppState {
    return this.#events.reduce(applyEvent, initialState);
  }

  // Time-travel: state at a specific point
  stateAt(timestamp: number): AppState {
    return this.#events
      .filter((e) => e.timestamp <= timestamp)
      .reduce(applyEvent, initialState);
  }

  // Serialize for persistence/debugging
  export() {
    return JSON.stringify(this.#events);
  }
  import(json: string) {
    this.#events = JSON.parse(json);
  }
}

function applyEvent(state: AppState, event: AppEvent): AppState {
  switch (event.type) {
    case "ITEM_ADDED_TO_CART":
      return { ...state, cart: [...state.cart, event.payload.item] };
    case "COUPON_APPLIED":
      return {
        ...state,
        coupon: event.payload.code,
        discount: event.payload.discount,
      };
    // ...
    default:
      return state;
  }
}
```

### Practical Event Sourcing: Undo/Redo

```typescript
class UndoableStore<TState, TEvent> {
  #events: TEvent[] = [];
  #undone: TEvent[] = [];
  #applyFn: (state: TState, event: TEvent) => TState;
  #initial: TState;

  constructor(
    initial: TState,
    applyFn: (state: TState, event: TEvent) => TState,
  ) {
    this.#initial = initial;
    this.#applyFn = applyFn;
  }

  dispatch(event: TEvent) {
    this.#events.push(event);
    this.#undone = []; // clear redo stack on new action
  }

  undo() {
    const last = this.#events.pop();
    if (last) this.#undone.push(last);
  }

  redo() {
    const next = this.#undone.pop();
    if (next) this.#events.push(next);
  }

  get state(): TState {
    return this.#events.reduce(this.#applyFn, this.#initial);
  }

  get canUndo() {
    return this.#events.length > 0;
  }
  get canRedo() {
    return this.#undone.length > 0;
  }
}
```

---

## 7. Reactive Streams with RxJS

RxJS models event streams as Observables — composable, cancellable sequences of events over time.

```typescript
import { fromEvent, merge, Subject } from "rxjs";
import {
  debounceTime,
  distinctUntilChanged,
  switchMap,
  map,
  catchError,
} from "rxjs/operators";

// Search input → live search results
const searchInput = document.getElementById("search") as HTMLInputElement;

const searchResults$ = fromEvent(searchInput, "input").pipe(
  map((event) => (event.target as HTMLInputElement).value),
  debounceTime(300), // wait 300ms after last keystroke
  distinctUntilChanged(), // skip if value didn't change
  switchMap(
    (
      query, // cancel previous request on new input
    ) =>
      from(searchApi.query(query)).pipe(
        catchError(() => of([])), // empty array on error
      ),
  ),
);

const subscription = searchResults$.subscribe((results) => {
  renderResults(results);
});

// Cleanup when component unmounts:
// subscription.unsubscribe();
```

### Combining Multiple Event Streams

```typescript
// Merge events from multiple sources into one stream
const cartUpdates$ = merge(
  fromEventBus("cart:item-added"),
  fromEventBus("cart:item-removed"),
  fromEventBus("cart:item-qty-changed"),
  fromWebSocket("/ws/cart"),
);

// Real-time price updates throttled to 2 per second
const priceUpdates$ = fromWebSocket("/ws/prices").pipe(
  throttleTime(500), // at most every 500ms
  map((msg) => msg.data),
  share(), // shared subscription for multiple observers
);

// Cart total that updates reactively
const cartTotal$ = combineLatest([cartItems$, priceUpdates$]).pipe(
  map(([items, prices]) =>
    items.reduce(
      (sum, item) =>
        sum + (prices[item.productId] ?? item.price) * item.quantity,
      0,
    ),
  ),
);
```

### Helper: fromEventBus

```typescript
// Bridge event bus to Observable
function fromEventBus<K extends keyof DomainEvents>(
  event: K,
): Observable<DomainEvents[K]> {
  return new Observable((subscriber) => {
    const unsubscribe = eventBus.subscribe(event, (data) => {
      subscriber.next(data);
    });
    return unsubscribe; // called on unsubscribe
  });
}
```

---

## 8. Events in React — Context, State, and Callbacks

React has its own event patterns that work within the component tree.

### Context as an Internal Event Channel

```typescript
// Context provides events within a component subtree
interface FormEvents {
  onFieldChange: (name: string, value: unknown) => void;
  onValidate:    (name: string) => void;
  onSubmit:      () => void;
}

const FormContext = createContext<FormEvents | null>(null);

function Form({ children, onSubmit }: FormProps) {
  const [values, setValues] = useState<Record<string, unknown>>({});
  const [errors, setErrors] = useState<Record<string, string>>({});

  const events: FormEvents = useMemo(() => ({
    onFieldChange: (name, value) => {
      setValues(v => ({ ...v, [name]: value }));
      setErrors(e => { const next = { ...e }; delete next[name]; return next; });
    },
    onValidate: (name) => {
      // run validation for field
    },
    onSubmit: () => onSubmit(values),
  }), [values, onSubmit]);

  return (
    <FormContext.Provider value={events}>
      {children}
    </FormContext.Provider>
  );
}

// Fields respond to form events
function FormField({ name, label }: FieldProps) {
  const { onFieldChange, onValidate } = useContext(FormContext)!;

  return (
    <input
      onChange={e => onFieldChange(name, e.target.value)}
      onBlur={() => onValidate(name)}
    />
  );
}
```

### Callback Chains (React's Direct Approach)

```typescript
// React's native approach: callback props
// Use when the call chain is predictable and shallow (< 3 levels)
function ProductCard({ product, onAddToCart, onWishlist }: ProductCardProps) {
  return (
    <div>
      <h3>{product.name}</h3>
      <button onClick={() => onAddToCart(product)}>Add to Cart</button>
      <button onClick={() => onWishlist(product)}>♡</button>
    </div>
  );
}

// Use event bus when:
//   - Propagation crosses more than 2-3 levels
//   - The responder is in a completely different part of the tree
//   - The emitter shouldn't know about the responder
```

---

## 9. Cross-Feature Communication with Events

The primary use case for event buses in large applications:

```typescript
// Feature isolation through events
// features/checkout/services/checkoutService.ts
import { eventBus } from "@/infrastructure/eventBus";

export async function completeCheckout(cart: Cart, payment: PaymentData) {
  const order = await checkoutApi.place({ cart, payment });

  // Publish: doesn't call any other feature directly
  eventBus.publish("order:placed", {
    orderId: order.id,
    total: order.total,
    itemCount: cart.items.length,
    paymentMethod: payment.method,
  });

  return order;
}

// features/notifications/subscriptions.ts
// (loaded at app startup)
eventBus.subscribe("order:placed", ({ orderId, total }) => {
  notificationService.show({
    type: "success",
    title: "Order Confirmed!",
    message: `Your order #${orderId} for $${total} has been placed.`,
    action: { label: "View Order", href: `/orders/${orderId}` },
  });
});

// infrastructure/analytics/subscriptions.ts
eventBus.subscribe("order:placed", (event) => {
  analytics.track("purchase", {
    transaction_id: event.orderId,
    value: event.total,
    items: event.itemCount,
  });
});

// features/cart/subscriptions.ts
eventBus.subscribe("order:placed", () => {
  cartStore.clear();
});

// features/loyalty/subscriptions.ts
eventBus.subscribe("order:placed", ({ orderId, total }) => {
  loyaltyService.awardPoints(total);
});
```

### Subscription Registration Pattern

```typescript
// app/subscriptions.ts — register all cross-feature subscriptions at startup
// Single file makes it clear what reacts to what

import "./features/notifications/subscriptions";
import "./features/analytics/subscriptions";
import "./infrastructure/realtime/subscriptions";
import "./features/cart/subscriptions";
import "./features/loyalty/subscriptions";

// app/index.tsx
import "./app/subscriptions"; // loaded before rendering
```

---

## 10. WebSocket Events and Real-Time Integration

```typescript
// infrastructure/realtime/RealtimeService.ts
class RealtimeService {
  #socket: WebSocket | null = null;
  #reconnectAttempts = 0;
  #maxReconnects = 5;

  connect(userId: string) {
    const wsUrl = `${config.wsUrl}?userId=${userId}&token=${getToken()}`;
    this.#socket = new WebSocket(wsUrl);

    this.#socket.onopen = () => {
      this.#reconnectAttempts = 0;
      console.info("[Realtime] Connected");
    };

    this.#socket.onmessage = ({ data }) => {
      try {
        const message: RealtimeMessage = JSON.parse(data);
        this.#dispatch(message);
      } catch (err) {
        console.error("[Realtime] Malformed message:", data);
      }
    };

    this.#socket.onclose = (event) => {
      if (!event.wasClean && this.#reconnectAttempts < this.#maxReconnects) {
        const delay = Math.min(1000 * 2 ** this.#reconnectAttempts, 30_000);
        this.#reconnectAttempts++;
        console.warn(`[Realtime] Disconnected. Reconnecting in ${delay}ms...`);
        setTimeout(() => this.connect(userId), delay);
      }
    };
  }

  #dispatch(message: RealtimeMessage) {
    // Bridge WebSocket messages to the application event bus
    switch (message.type) {
      case "ORDER_STATUS_CHANGED":
        eventBus.publish("order:status-changed", message.payload);
        break;
      case "NOTIFICATION":
        eventBus.publish("ui:notification-shown", message.payload);
        break;
      case "PRICE_UPDATE":
        // High-frequency: don't go through event bus — directly update store
        priceStore.update(message.payload);
        break;
    }
  }

  disconnect() {
    this.#socket?.close(1000, "User logged out");
    this.#socket = null;
  }
}

export const realtimeService = new RealtimeService();
```

---

## 11. Event Logging and Analytics

Event buses are natural integration points for analytics tracking.

```typescript
// infrastructure/analytics/analyticsMiddleware.ts
// Intercept all events for analytics

const TRACKED_EVENTS: (keyof DomainEvents)[] = [
  "auth:login-success",
  "cart:item-added",
  "cart:checkout-started",
  "order:placed",
];

// Middleware pattern: wrap the event bus
function createAnalyticsMiddleware(eventBus: EventBus<DomainEvents>) {
  TRACKED_EVENTS.forEach((event) => {
    eventBus.subscribe(event, (data) => {
      analytics.track(String(event), {
        ...data,
        timestamp: Date.now(),
        sessionId: getSessionId(),
        userId: getCurrentUserId(),
      });
    });
  });
}

// Or: generic event interceptor
class InstrumentedEventBus<
  T extends Record<string, unknown>,
> extends EventBus<T> {
  #logger: (event: string, data: unknown) => void;

  constructor(logger: (event: string, data: unknown) => void) {
    super();
    this.#logger = logger;
  }

  override publish<K extends keyof T>(event: K, data: T[K]): void {
    this.#logger(String(event), data);
    super.publish(event, data);
  }
}
```

---

## 12. Testing Event-Driven Code

```typescript
// Testing event publishers
describe("checkoutService", () => {
  it("publishes order:placed event after successful checkout", async () => {
    // Arrange
    const handler = jest.fn();
    const unsubscribe = eventBus.subscribe("order:placed", handler);
    server.use(
      http.post("/api/checkout", () =>
        HttpResponse.json({ id: "order-123", total: 99.99 }),
      ),
    );

    // Act
    await checkoutService.completeCheckout(mockCart, mockPayment);

    // Assert
    expect(handler).toHaveBeenCalledWith({
      orderId: "order-123",
      total: 99.99,
      itemCount: mockCart.items.length,
      paymentMethod: mockPayment.method,
    });

    // Cleanup
    unsubscribe();
  });
});

// Testing event subscribers
describe("loyaltyService", () => {
  it("awards points when order is placed", () => {
    // Arrange
    const awardPoints = jest.spyOn(loyaltyApi, "awardPoints");

    // Act: simulate the event as if checkout service published it
    eventBus.publish("order:placed", {
      orderId: "test-order",
      total: 50,
      itemCount: 2,
      paymentMethod: "card",
    });

    // Assert: subscriber reacted
    expect(awardPoints).toHaveBeenCalledWith(50);
  });
});

// Testing with a test event bus (isolated)
describe("notificationsSubscription", () => {
  let testBus: EventBus<DomainEvents>;

  beforeEach(() => {
    testBus = new EventBus<DomainEvents>();
    // Register the subscription under test with the test bus
    registerNotificationSubscriptions(testBus);
  });

  it("shows success notification on order placed", () => {
    const show = jest.spyOn(notificationService, "show");

    testBus.publish("order:placed", {
      orderId: "42",
      total: 29.99,
      itemCount: 1,
      paymentMethod: "card",
    });

    expect(show).toHaveBeenCalledWith(
      expect.objectContaining({
        type: "success",
        title: "Order Confirmed!",
      }),
    );
  });
});
```

---

## 13. Good Practices

### ✅ Keep event names in past tense (things that happened)

```typescript
// ✅ Events describe completed actions (past tense)
"cart:item-added"; // ✓
"order:placed"; // ✓
"user:password-changed"; // ✓

// ❌ Commands disguised as events (present tense / imperative)
"cart:add-item"; // ✗ — this is a command
"send-welcome-email"; // ✗ — this is a command
"refresh-orders"; // ✗ — this is a command
```

### ✅ Include enough context in events for all subscribers

```typescript
// ✅ Event contains sufficient data for all likely subscribers
eventBus.publish("cart:item-added", {
  productId: product.id,
  productName: product.name, // ← analytics needs this
  price: product.price, // ← price tracker needs this
  quantity: 1,
  source: "pdp", // ← attribution needs this
  cartId: cart.id, // ← sync needs this
});

// ❌ Thin event forces subscribers to do extra fetches
eventBus.publish("cart:item-added", { productId }); // subscriber must fetch product details
```

### ✅ Always clean up subscriptions

```typescript
// ✅ In React: cleanup in useEffect
useEffect(() => {
  const unsubscribe = eventBus.subscribe("order:placed", handleOrderPlaced);
  return unsubscribe; // called on unmount
}, []);

// ✅ Store subscriptions and cleanup in service destroy
class FeatureService {
  #subscriptions: Unsubscribe[] = [];

  init() {
    this.#subscriptions.push(
      eventBus.subscribe("auth:logout", this.#handleLogout.bind(this)),
      eventBus.subscribe("user:updated", this.#handleUserUpdate.bind(this)),
    );
  }

  destroy() {
    this.#subscriptions.forEach((unsub) => unsub());
    this.#subscriptions = [];
  }
}
```

---

## 14. Bad Practices

### ❌ Using events for synchronous request/response

```typescript
// ❌ Events for synchronous operations — awkward and error-prone
eventBus.publish("cart:get-total-request", { cartId });
// ... hoping someone sets the response somewhere ...
const total = cartStore.lastTotal; // timing-dependent!

// ✅ Direct call for synchronous data access
const total = cartService.getTotal(cartId);
```

### ❌ Deeply nested event chains

```typescript
// ❌ Events triggering events triggering events — hard to trace
eventBus.subscribe('user:created', () => {
  eventBus.publish('email:send-welcome', ...);
});
eventBus.subscribe('email:send-welcome', () => {
  eventBus.publish('analytics:signup', ...);
});
eventBus.subscribe('analytics:signup', () => {
  eventBus.publish('onboarding:start', ...);
});
// What happens when onboarding:start fails? Hard to debug, hard to reason about.

// ✅ Explicit chain or use the original event for independent reactions
eventBus.subscribe('user:created', handleWelcomeEmail);
eventBus.subscribe('user:created', handleAnalytics);
eventBus.subscribe('user:created', handleOnboarding);
```

### ❌ Global event bus for in-component communication

```typescript
// ❌ Event bus for parent-child communication
// Child component publishes to global bus
eventBus.publish('modal:close-button-clicked', {});

// Parent subscribes
useEffect(() => {
  return eventBus.subscribe('modal:close-button-clicked', () => setOpen(false));
}, []);

// ✅ Callback prop — simpler, more direct
<Modal onClose={() => setOpen(false)} />
```

---

## 15. Common Mistakes

### Mistake 1 — Forgetting to remove listeners

```typescript
// ❌ Subscription created on every render, never cleaned up
function Component() {
  eventBus.subscribe("cart:updated", updateDisplay); // adds a new listener every render!
  // After 100 renders: 100 redundant listeners
}

// ✅ Subscribe once in useEffect with cleanup
function Component() {
  useEffect(() => {
    return eventBus.subscribe("cart:updated", updateDisplay); // cleanup on unmount
  }, []);
}
```

### Mistake 2 — Events with side effects in listeners that shouldn't happen

```typescript
// ❌ Subscribe once, forget it changes behavior system-wide
// In tests: event bus is a singleton, subscriptions leak between tests

// ✅ Use isolated event bus per test suite
beforeEach(() => {
  eventBus.unsubscribeAll();
  // Or: create a new EventBus instance for each test
});
```

### Mistake 3 — Publishing events before subscribers are registered

```typescript
// ❌ App initialization: event published before subscriber is set up
eventBus.publish("app:ready", {}); // subscribers not registered yet!
initializeSubscriptions(); // too late

// ✅ Initialize subscriptions first, then trigger events
initializeSubscriptions();
eventBus.publish("app:ready", {});

// Or: use a "behavior" pattern — new subscribers receive the last value
class BehaviorEvent<T> {
  #lastValue: T | undefined;
  #bus: EventBus;

  publish(data: T) {
    this.#lastValue = data;
    this.#bus.publish(this.#key, data);
  }

  subscribe(handler: (data: T) => void): Unsubscribe {
    const unsub = this.#bus.subscribe(this.#key, handler);
    if (this.#lastValue !== undefined) handler(this.#lastValue); // replay last value
    return unsub;
  }
}
```

---

## 16. Interview-Level Explanation

> **"What is event-driven architecture on the frontend? When would you use it over direct module calls?"**

**Strong answer:**

> "Event-driven architecture replaces direct function calls between modules with a publish/subscribe model. Instead of Module A calling Module B directly, Module A publishes an event — 'checkout completed' — and any number of modules subscribe to react: notifications, analytics, order refresh, loyalty points. Module A doesn't know or care about any of them.
>
> The key architectural benefit is decoupling. With direct calls, the cart service must import and call the notifications service, the analytics service, and every other module that needs to react. This creates a dependency graph where the cart can't be tested or changed without all its dependencies. With events, the cart emits one event and knows nothing about what happens next. Each subscriber is independently added, removed, or changed without touching the cart.
>
> The practical implementation is an event bus — a typed pub/sub system. I model event names as past-tense domain events: 'user:registered', 'order:placed', 'payment:failed'. TypeScript discriminated unions enforce that each event name has a specific payload shape, catching mismatches at compile time.
>
> Events work best for cross-feature or cross-module reactions. Within a React component tree, callback props are usually the right choice — they're simpler, more predictable, and TypeScript handles them natively. Events make sense when the propagation crosses feature boundaries, when multiple unrelated modules react to the same occurrence, or in micro-frontend architectures where modules literally can't share direct imports.
>
> The common failure mode is using events for everything, including synchronous queries ('get the current cart total') or parent-child communication that's three levels deep. Events are asynchronous by nature and fire-and-forget — you lose the return value and the call stack. For any case where you need a response, use a direct call. Events are for 'this happened, anyone who cares can react.'"

---

## 17. Exercises

### Exercise 1 — Design an event system

A user completes a purchase. Design:

1. The event type definition
2. The publisher code in the checkout service
3. Three independent subscribers (you choose which features react)
4. How to register and clean up subscriptions at app startup

<details>
<summary>Solution</summary>

```typescript
// 1. Event type definition
interface DomainEvents {
  "purchase:completed": {
    orderId: string;
    userId: string;
    total: number;
    items: Array<{
      productId: string;
      name: string;
      price: number;
      qty: number;
    }>;
    paymentMethod: "card" | "paypal" | "apple-pay" | "google-pay";
    couponCode?: string;
    timestamp: number;
  };
}

// 2. Publisher (checkout service)
// features/checkout/checkoutService.ts
async function completePurchase(checkoutData: CheckoutData): Promise<Order> {
  const order = await checkoutApi.place(checkoutData);

  eventBus.publish("purchase:completed", {
    orderId: order.id,
    userId: currentUser.id,
    total: order.total,
    items: order.items,
    paymentMethod: checkoutData.payment.method,
    couponCode: checkoutData.coupon?.code,
    timestamp: Date.now(),
  });

  return order;
}

// 3. Three subscribers

// features/notifications/subscriptions.ts
export function registerNotificationSubscriptions() {
  return eventBus.subscribe("purchase:completed", ({ orderId, total }) => {
    notificationService.show({
      type: "success",
      title: "Purchase Confirmed!",
      message: `Order #${orderId} · $${total.toFixed(2)}`,
      link: { label: "View Order", href: `/orders/${orderId}` },
      ttl: 8000,
    });
  });
}

// infrastructure/analytics/subscriptions.ts
export function registerAnalyticsSubscriptions() {
  return eventBus.subscribe("purchase:completed", (event) => {
    analytics.track("purchase", {
      transaction_id: event.orderId,
      value: event.total,
      payment_type: event.paymentMethod,
      coupon: event.couponCode,
      items: event.items.map((i) => ({
        item_id: i.productId,
        item_name: i.name,
        price: i.price,
        quantity: i.qty,
      })),
    });
  });
}

// features/cart/subscriptions.ts
export function registerCartSubscriptions() {
  return eventBus.subscribe("purchase:completed", () => {
    cartStore.clear();
    queryClient.invalidateQueries({ queryKey: ["cart"] });
  });
}

// 4. Registration at app startup
// app/subscriptions.ts
import { registerNotificationSubscriptions } from "@/features/notifications/subscriptions";
import { registerAnalyticsSubscriptions } from "@/infrastructure/analytics/subscriptions";
import { registerCartSubscriptions } from "@/features/cart/subscriptions";

const unsubscribers: Unsubscribe[] = [];

export function initSubscriptions() {
  unsubscribers.push(
    registerNotificationSubscriptions(),
    registerAnalyticsSubscriptions(),
    registerCartSubscriptions(),
  );
}

export function destroySubscriptions() {
  unsubscribers.forEach((fn) => fn());
  unsubscribers.length = 0;
}

// app/index.tsx
initSubscriptions(); // before rendering
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md) — Observer pattern fundamentals
- [`javascript-core/15-pub-sub-systems.md`](../javascript-core/15-pub-sub-systems.md) — Full pub/sub implementation
- [`system-design/03-micro-frontends.md`](./03-micro-frontends.md) — Events for cross-MFE communication
- [`system-design/02-feature-based-structure.md`](./02-feature-based-structure.md) — Feature isolation enabled by events
- [`patterns/01-observer.md`](../patterns/01-observer.md) — Observer pattern in depth

---

<div align="center">

**Next:** [`system-design/07-plugin-systems.md`](./07-plugin-systems.md) →

</div>
