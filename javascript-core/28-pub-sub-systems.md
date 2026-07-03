# 15 — Pub/Sub Systems

> **"Pub/Sub is what happens when you take the Observer pattern and add an intermediary. That intermediary — the event bus — is what makes large-scale frontend architecture possible. Modules that have never heard of each other can communicate through shared channels without creating dependencies."**

Pub/Sub (Publish/Subscribe) is the architectural backbone of event-driven frontend systems. It powers real-time dashboards, cross-component communication, plugin architectures, and micro-frontend coordination. This document builds a complete, production-grade event bus from scratch — covering synchronous and asynchronous messaging, typed channels, replay, middleware, and the patterns that make large-scale frontend applications maintainable.

---

## 📚 Table of Contents

1. [Pub/Sub vs Observer — Why It Matters](#1-pubsub-vs-observer--why-it-matters)
2. [The Event Bus — Core Architecture](#2-the-event-bus--core-architecture)
3. [Building a Production Event Bus](#3-building-a-production-event-bus)
4. [Typed Event Channels](#4-typed-event-channels)
5. [Async Pub/Sub](#5-async-pubsub)
6. [Message Replay and History](#6-message-replay-and-history)
7. [Middleware Pipeline](#7-middleware-pipeline)
8. [Priority Subscribers](#8-priority-subscribers)
9. [Wildcards and Pattern Matching](#9-wildcards-and-pattern-matching)
10. [Cross-Module Communication Patterns](#10-cross-module-communication-patterns)
11. [Pub/Sub in Micro-Frontend Architecture](#11-pubsub-in-micro-frontend-architecture)
12. [Request/Reply Pattern](#12-requestreply-pattern)
13. [Dead Letter Queue](#13-dead-letter-queue)
14. [Testing Pub/Sub Systems](#14-testing-pubsub-systems)
15. [Memory Management](#15-memory-management)
16. [Good Practices](#16-good-practices)
17. [Bad Practices](#17-bad-practices)
18. [Common Mistakes](#18-common-mistakes)
19. [Interview-Level Explanation](#19-interview-level-explanation)
20. [Exercises](#20-exercises)

---

## 1. Pub/Sub vs Observer — Why It Matters

The Observer pattern requires the subject to hold direct references to observers. Pub/Sub inserts a **message broker** between publishers and subscribers — neither knows the other exists.

```
OBSERVER:
  Counter (Subject) ──► Dashboard (Observer)
  Counter (Subject) ──► Logger (Observer)
  Counter knows about Dashboard and Logger.
  If Counter changes API, Dashboard and Logger break.

PUB/SUB:
  Counter ──publish('count:changed', data)──► [EventBus] ◄──subscribe('count:changed')── Dashboard
                                                          ◄──subscribe('count:changed')── Logger
  Counter knows nothing about Dashboard or Logger.
  Dashboard knows nothing about Counter.
  Modules are completely decoupled.
```

### When to Use Pub/Sub Over Observer

```
Use Observer when:
  - One object (subject) needs to directly notify specific dependents
  - The relationship is explicit and known at compile time
  - Performance is critical (no intermediary overhead)
  - Examples: reactive store, component state, data binding

Use Pub/Sub when:
  - Modules need to communicate without knowing about each other
  - Dynamic plugin/extension architecture
  - Cross-module or cross-team communication
  - Micro-frontend coordination
  - You want to add/remove subscribers without changing publishers
  - Examples: analytics events, feature flags, UI notifications, cross-tab sync
```

---

## 2. The Event Bus — Core Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        EVENT BUS                              │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Channel Registry: Map<channel, Set<subscriber>>     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  publish(channel, data)                                      │
│    → find all subscribers for channel                        │
│    → call each subscriber with data                          │
│                                                               │
│  subscribe(channel, handler) → unsubscribe fn               │
│    → add handler to channel's subscriber set                 │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Publishers    │ Channel │    Subscribers             │   │
│  │  ─────────────────────────────────────────────────── │   │
│  │  UserModule    │ user:*  │ Analytics, Logger          │   │
│  │  CartModule    │ cart:*  │ Checkout, Recommendations  │   │
│  │  APIModule     │ api:*   │ Cache, Retry, Loading UI   │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Core Properties of a Good Event Bus

```
1. Decoupling     — publishers and subscribers never reference each other
2. One-to-many    — one publish → many subscribers
3. Async-safe     — subscriptions can be added before or after publish
4. Memory-safe    — unsubscribe must cleanly remove all references
5. Error isolation — one failing subscriber doesn't break others
6. Type-safe      — channels have typed payloads (in TypeScript)
7. Debuggable     — events can be logged, replayed, traced
```

---

## 3. Building a Production Event Bus

### The Core Implementation

```javascript
class EventBus {
  #channels = new Map(); // channel → Set<subscriber>
  #history = new Map(); // channel → circular buffer of past events
  #middleware = []; // global middleware pipeline
  #historySize;
  #debug;

  constructor({ historySize = 0, debug = false } = {}) {
    this.#historySize = historySize;
    this.#debug = debug;
  }

  // ── Subscribe ──────────────────────────────────────────────────

  /**
   * Subscribe to a channel.
   * @returns unsubscribe function
   */
  on(channel, handler, options = {}) {
    this.#validateChannel(channel);
    this.#validateHandler(handler);

    if (!this.#channels.has(channel)) {
      this.#channels.set(channel, new Map());
    }

    const subscribers = this.#channels.get(channel);

    // Wrap handler with metadata
    const subscription = {
      handler,
      priority: options.priority ?? 0,
      once: options.once ?? false,
      id: Symbol(channel),
    };

    subscribers.set(subscription.id, subscription);

    if (this.#debug) {
      console.log(
        `[EventBus] subscribed to "${channel}" (total: ${subscribers.size})`,
      );
    }

    // If history replay requested: immediately call with historical events
    if (options.replay && this.#historySize > 0) {
      const past = this.#history.get(channel) ?? [];
      past.forEach((event) => {
        try {
          handler(event.data, event.meta);
        } catch (err) {
          this.#handleError(err, channel, event.data);
        }
      });
    }

    // Return unsubscribe function
    return () => {
      subscribers.delete(subscription.id);
      if (subscribers.size === 0) this.#channels.delete(channel);
      if (this.#debug) {
        console.log(`[EventBus] unsubscribed from "${channel}"`);
      }
    };
  }

  /**
   * Subscribe to a channel, auto-unsubscribes after first message.
   */
  once(channel, handler, options = {}) {
    return this.on(channel, handler, { ...options, once: true });
  }

  // ── Publish ────────────────────────────────────────────────────

  /**
   * Publish data to a channel.
   * @returns number of subscribers notified
   */
  publish(channel, data, meta = {}) {
    this.#validateChannel(channel);

    const event = {
      channel,
      data,
      meta: {
        timestamp: Date.now(),
        id: Math.random().toString(36).slice(2),
        ...meta,
      },
    };

    // Run through middleware pipeline
    const processedEvent = this.#runMiddleware(event);
    if (!processedEvent) return 0; // middleware blocked the event

    // Store in history
    this.#recordHistory(channel, processedEvent);

    // Notify subscribers (sorted by priority, highest first)
    const subscribers = this.#channels.get(channel);
    if (!subscribers || subscribers.size === 0) {
      if (this.#debug)
        console.log(`[EventBus] no subscribers for "${channel}"`);
      return 0;
    }

    const sorted = [...subscribers.values()].sort(
      (a, b) => b.priority - a.priority,
    );
    let notified = 0;

    for (const subscription of sorted) {
      try {
        subscription.handler(processedEvent.data, processedEvent.meta);
        notified++;

        if (subscription.once) {
          subscribers.delete(subscription.id);
        }
      } catch (err) {
        this.#handleError(err, channel, processedEvent.data);
      }
    }

    if (this.#debug) {
      console.log(
        `[EventBus] published "${channel}" → ${notified} subscribers`,
      );
    }

    return notified;
  }

  // ── Utilities ──────────────────────────────────────────────────

  /**
   * Add a middleware function that processes all events.
   * Return null/undefined to block the event.
   */
  use(middlewareFn) {
    this.#middleware.push(middlewareFn);
    return () => {
      const idx = this.#middleware.indexOf(middlewareFn);
      if (idx !== -1) this.#middleware.splice(idx, 1);
    };
  }

  /**
   * Get the event history for a channel.
   */
  getHistory(channel) {
    return [...(this.#history.get(channel) ?? [])];
  }

  /**
   * Clear all subscribers for a channel, or all channels.
   */
  clear(channel) {
    if (channel) {
      this.#channels.delete(channel);
      this.#history.delete(channel);
    } else {
      this.#channels.clear();
      this.#history.clear();
    }
  }

  /**
   * Get stats about current subscriptions.
   */
  stats() {
    const result = {};
    for (const [channel, subs] of this.#channels) {
      result[channel] = subs.size;
    }
    return result;
  }

  // ── Private ────────────────────────────────────────────────────

  #validateChannel(channel) {
    if (typeof channel !== "string" || channel.length === 0) {
      throw new TypeError("Channel must be a non-empty string");
    }
  }

  #validateHandler(handler) {
    if (typeof handler !== "function") {
      throw new TypeError("Handler must be a function");
    }
  }

  #runMiddleware(event) {
    let current = event;
    for (const mw of this.#middleware) {
      current = mw(current);
      if (!current) return null; // blocked
    }
    return current;
  }

  #recordHistory(channel, event) {
    if (this.#historySize === 0) return;

    if (!this.#history.has(channel)) {
      this.#history.set(channel, []);
    }

    const history = this.#history.get(channel);
    history.push(event);

    // Trim to max size (circular buffer behavior)
    if (history.length > this.#historySize) {
      history.shift();
    }
  }

  #handleError(err, channel, data) {
    console.error(`[EventBus] Error in subscriber for "${channel}":`, err);
    // Publish to error channel (won't loop — error handlers don't throw)
    if (channel !== "bus:error") {
      this.publish("bus:error", { error: err, channel, data });
    }
  }
}

// ─── Create singleton for app use ──────────────────────────────
const bus = new EventBus({ historySize: 10, debug: false });
export default bus;
```

---

## 4. Typed Event Channels

For large codebases, define channel schemas to catch mistakes at development time:

```javascript
/**
 * Channel schema registry — define what data each channel carries.
 * In TypeScript, this provides full type inference.
 */
class TypedEventBus extends EventBus {
  #schemas = new Map(); // channel → validator function

  /**
   * Register a channel with a validation schema.
   */
  defineChannel(channel, validator) {
    this.#schemas.set(channel, validator);
    return this;
  }

  publish(channel, data, meta) {
    const validator = this.#schemas.get(channel);
    if (validator) {
      const result = validator(data);
      if (result !== true) {
        throw new TypeError(
          `Invalid payload for channel "${channel}": ${result}`,
        );
      }
    }
    return super.publish(channel, data, meta);
  }
}

// Usage
const bus = new TypedEventBus();

// Define channel schemas (validators)
bus.defineChannel("user:login", (data) => {
  if (typeof data?.userId !== "string") return "userId must be a string";
  if (typeof data?.timestamp !== "number") return "timestamp must be a number";
  return true;
});

bus.defineChannel("cart:item:added", (data) => {
  if (!data?.productId) return "productId is required";
  if (typeof data?.quantity !== "number" || data.quantity < 1)
    return "quantity must be >= 1";
  return true;
});

// ✅ Valid publish
bus.publish("user:login", { userId: "42", timestamp: Date.now() });

// ❌ Throws at dev time
bus.publish("user:login", { userId: 42 }); // TypeError: userId must be a string
```

---

## 5. Async Pub/Sub

Sometimes subscribers need to do async work. The standard pub/sub notifies all subscribers synchronously — for async work, you need a different approach.

### Async Publish — Fire and Forget

```javascript
// Publish returns immediately; async handlers run in background
class AsyncEventBus extends EventBus {
  publishAsync(channel, data) {
    const subscribers = this.getSubscribers(channel);

    // Fire all async handlers concurrently, don't await
    subscribers.forEach(async (handler) => {
      try {
        await handler(data);
      } catch (err) {
        console.error(`Async handler error on "${channel}":`, err);
      }
    });
  }
}
```

### Async Publish — Wait for All Handlers

```javascript
// Await all async subscribers before continuing
class AwaitableEventBus extends EventBus {
  async publishAndWait(channel, data) {
    const subscribers = [...(this.getSubscribersRaw(channel) ?? [])];

    const results = await Promise.allSettled(
      subscribers.map((sub) => Promise.resolve(sub.handler(data))),
    );

    const failures = results.filter((r) => r.status === "rejected");
    if (failures.length > 0) {
      console.error(
        `${failures.length} async handlers failed for "${channel}"`,
      );
    }

    return results;
  }
}
```

### Async Queue Per Channel

```javascript
// Each channel has its own async queue — messages processed in order
class QueuedEventBus {
  #queues = new Map(); // channel → AsyncQueue
  #subscribers = new Map();

  subscribe(channel, handler) {
    if (!this.#subscribers.has(channel))
      this.#subscribers.set(channel, new Set());
    this.#subscribers.get(channel).add(handler);
    return () => this.#subscribers.get(channel)?.delete(handler);
  }

  async publish(channel, data) {
    if (!this.#queues.has(channel)) {
      this.#queues.set(channel, this.#createQueue());
    }
    return this.#queues
      .get(channel)
      .enqueue(() => this.#dispatch(channel, data));
  }

  async #dispatch(channel, data) {
    const handlers = this.#subscribers.get(channel);
    if (!handlers) return;

    for (const handler of handlers) {
      await handler(data); // sequential processing per channel
    }
  }

  #createQueue() {
    let running = false;
    const queue = [];

    return {
      enqueue(task) {
        return new Promise((resolve, reject) => {
          queue.push({ task, resolve, reject });
          if (!running) this.flush();
        });
      },
      async flush() {
        running = true;
        while (queue.length > 0) {
          const { task, resolve, reject } = queue.shift();
          try {
            resolve(await task());
          } catch (err) {
            reject(err);
          }
        }
        running = false;
      },
    };
  }
}
```

---

## 6. Message Replay and History

Some systems need new subscribers to receive past messages — especially for initialization state or missed events.

```javascript
// Bus with replay support (built into Section 3's EventBus)
const bus = new EventBus({ historySize: 50 }); // keep last 50 events per channel

// Publisher: sends configuration whenever it changes
bus.publish("config:loaded", { theme: "dark", locale: "en-US" });

// Subscriber that joins AFTER the publish — gets replayed
setTimeout(() => {
  bus.on(
    "config:loaded",
    (config) => {
      applyConfig(config); // receives the previous 'config:loaded' event
    },
    { replay: true },
  ); // ← replay: true triggers history replay on subscribe
}, 5000);
```

### BehaviorSubject Pattern — Always Has a Current Value

```javascript
// Like RxJS BehaviorSubject — new subscribers always get the latest value
class BehaviorChannel {
  #bus;
  #channel;
  #currentValue;

  constructor(bus, channel, initialValue) {
    this.#bus = bus;
    this.#channel = channel;
    this.#currentValue = initialValue;
  }

  publish(value) {
    this.#currentValue = value;
    this.#bus.publish(this.#channel, value);
  }

  subscribe(handler) {
    // Immediately call with current value
    handler(this.#currentValue);
    // Then subscribe for future updates
    return this.#bus.on(this.#channel, handler);
  }

  get current() {
    return this.#currentValue;
  }
}

// Usage
const themeChannel = new BehaviorChannel(bus, "ui:theme", "light");

themeChannel.publish("dark"); // notifies existing subscribers

// New subscriber immediately receives 'dark' — no missed message
themeChannel.subscribe((theme) => applyTheme(theme)); // receives 'dark' immediately
```

---

## 7. Middleware Pipeline

Middleware allows you to intercept, transform, log, or block events before they reach subscribers.

```javascript
// Example middleware: logging
const loggingMiddleware = (event) => {
  console.log(`[${new Date().toISOString()}] ${event.channel}:`, event.data);
  return event; // pass through unchanged
};

// Example middleware: authentication guard
const authMiddleware = (event) => {
  if (event.channel.startsWith("admin:") && !isAuthenticated()) {
    console.warn(`Blocked unauthorized event: ${event.channel}`);
    return null; // block the event
  }
  return event;
};

// Example middleware: event transformation
const timestampMiddleware = (event) => ({
  ...event,
  meta: {
    ...event.meta,
    processedAt: performance.now(),
  },
});

// Example middleware: rate limiting per channel
function createRateLimitMiddleware(maxPerSecond) {
  const counts = new Map();

  return (event) => {
    const now = Date.now();
    const key = event.channel;
    const record = counts.get(key) ?? { count: 0, reset: now + 1000 };

    if (now > record.reset) {
      record.count = 0;
      record.reset = now + 1000;
    }

    record.count++;
    counts.set(key, record);

    if (record.count > maxPerSecond) {
      console.warn(
        `Rate limit exceeded for "${event.channel}" (${record.count}/s)`,
      );
      return null; // block
    }

    return event;
  };
}

// Register middleware
const bus = new EventBus({ debug: true });
bus.use(loggingMiddleware);
bus.use(authMiddleware);
bus.use(timestampMiddleware);
bus.use(createRateLimitMiddleware(100));
```

---

## 8. Priority Subscribers

When multiple subscribers handle the same event, ordering matters. Priority ensures critical handlers (security, auth) run before UI handlers.

```javascript
// Priority levels
const PRIORITY = {
  CRITICAL: 1000, // security, auth guards
  HIGH: 100, // data processing
  NORMAL: 0, // default
  LOW: -100, // analytics, logging
};

// Subscribe with priority
const unsubAuth = bus.on("form:submit", validateAndBlock, {
  priority: PRIORITY.CRITICAL, // runs first
});

const unsubProcess = bus.on("form:submit", processFormData, {
  priority: PRIORITY.HIGH, // runs second
});

const unsubAnalytics = bus.on("form:submit", trackFormSubmission, {
  priority: PRIORITY.LOW, // runs last
});

// If validateAndBlock rejects the form data, it can set a flag
// that processFormData checks — or the bus can support cancellation:
```

### Cancellable Events

```javascript
class CancellableEventBus extends EventBus {
  publish(channel, data, meta = {}) {
    const event = { channel, data, meta, cancelled: false };

    // Cancel function passed to each subscriber
    const cancel = () => {
      event.cancelled = true;
    };

    const subscribers = this.#getSortedSubscribers(channel);
    for (const sub of subscribers) {
      if (event.cancelled) break; // stop notifying if cancelled
      try {
        sub.handler(event.data, event.meta, cancel);
      } catch (err) {
        console.error(err);
      }
    }

    return !event.cancelled;
  }
}

// Usage
bus.on(
  "form:submit",
  (data, meta, cancel) => {
    if (!isValid(data)) {
      cancel(); // prevent further subscribers from running
      showError("Invalid form data");
    }
  },
  { priority: PRIORITY.CRITICAL },
);

bus.on("form:submit", (data) => {
  submitToServer(data); // won't run if previous subscriber cancelled
});
```

---

## 9. Wildcards and Pattern Matching

Large systems benefit from subscribing to patterns rather than individual channels.

```javascript
class WildcardEventBus extends EventBus {
  #wildcardSubscribers = []; // [{ pattern, handler }]

  on(channel, handler, options = {}) {
    // If channel contains wildcard — register as pattern
    if (channel.includes("*") || channel.includes("?")) {
      const regex = this.#patternToRegex(channel);
      const unsub = () => {
        const idx = this.#wildcardSubscribers.findIndex(
          (s) => s.regex === regex && s.handler === handler,
        );
        if (idx !== -1) this.#wildcardSubscribers.splice(idx, 1);
      };
      this.#wildcardSubscribers.push({ regex, pattern: channel, handler });
      return unsub;
    }
    return super.on(channel, handler, options);
  }

  publish(channel, data, meta) {
    // Notify exact subscribers
    const count = super.publish(channel, data, meta);

    // Notify wildcard subscribers whose pattern matches
    for (const { regex, handler } of this.#wildcardSubscribers) {
      if (regex.test(channel)) {
        try {
          handler(data, { ...meta, channel });
        } catch (err) {
          console.error(err);
        }
      }
    }

    return count;
  }

  #patternToRegex(pattern) {
    const escaped = pattern
      .replace(/[.+^${}()|[\]\\]/g, "\\$&") // escape regex special chars
      .replace(/\*/g, ".*") // * → match anything
      .replace(/\?/g, "."); // ? → match single char
    return new RegExp(`^${escaped}$`);
  }
}

// Usage
const bus = new WildcardEventBus();

// Subscribe to all user events
bus.on("user:*", (data, meta) => {
  analytics.track(meta.channel, data);
});

// Subscribe to all events
bus.on("*", (data, meta) => {
  devLogger.log(meta.channel, data);
});

bus.publish("user:login", { userId: "1" }); // matches 'user:*' and '*'
bus.publish("user:logout", { userId: "1" }); // matches 'user:*' and '*'
bus.publish("cart:updated", { total: 99 }); // matches '*' only
```

---

## 10. Cross-Module Communication Patterns

### Pattern 1 — Domain Events

Modules publish domain events; other modules subscribe. No direct coupling.

```javascript
// userModule.js
import bus from "./eventBus.js";

export async function login(credentials) {
  const user = await authApi.login(credentials);

  // Publish domain event — user module doesn't know who cares
  bus.publish("user:authenticated", {
    userId: user.id,
    role: user.role,
    timestamp: Date.now(),
  });

  return user;
}

// analyticsModule.js — completely separate file/team
import bus from "./eventBus.js";

bus.on("user:authenticated", ({ userId }) => {
  analytics.identify(userId);
  analytics.track("login");
});

// permissionsModule.js — also completely separate
import bus from "./eventBus.js";

bus.on("user:authenticated", ({ role }) => {
  loadPermissions(role);
  updateNavigation(role);
});

// Both respond to the same event — UserModule knows nothing about them
```

### Pattern 2 — Command Events

Commands are intent-driven messages: "I want X to happen."

```javascript
// UI publishes a command
bus.publish("cart:item:add", { productId: "abc", quantity: 2 });

// CartModule handles the command
bus.on("cart:item:add", ({ productId, quantity }) => {
  addToCart(productId, quantity);
  bus.publish("cart:updated", { items: getCartItems() }); // result event
});

// Notifications module reacts to the result
bus.on("cart:updated", ({ items }) => {
  updateCartBadge(items.length);
});
```

### Pattern 3 — Feature Flags via Events

```javascript
// Feature flag changes broadcast to all modules
const featureFlagBus = new BehaviorChannel(bus, "feature:flags", {});

// When flags change on the server, push to bus
async function refreshFlags() {
  const flags = await fetchFeatureFlags();
  featureFlagBus.publish(flags);
}

// Any module can subscribe to flag changes
featureFlagBus.subscribe((flags) => {
  if (flags.newCheckout) {
    swapCheckoutModule();
  }
});
```

---

## 11. Pub/Sub in Micro-Frontend Architecture

In micro-frontend systems, different apps need to coordinate without sharing code directly.

```javascript
// shell/eventBus.js — the shared event bus exposed on window
// (accessible to all micro-frontends)
window.__APP_EVENT_BUS__ = new EventBus({ historySize: 20, debug: false });

// ─── Micro-Frontend A (Navigation App) ────────────────────────
const bus = window.__APP_EVENT_BUS__;

bus.publish("nav:route:changed", {
  from: "/dashboard",
  to: "/settings",
  params: { tab: "profile" },
});

// ─── Micro-Frontend B (Settings App) ──────────────────────────
const bus = window.__APP_EVENT_BUS__;

bus.on("nav:route:changed", ({ to, params }) => {
  if (to === "/settings") {
    loadSettingsTab(params.tab);
  }
});

// ─── Micro-Frontend C (Analytics App) ─────────────────────────
const bus = window.__APP_EVENT_BUS__;

bus.on("nav:route:changed", ({ from, to }) => {
  analytics.pageView({ from, to });
});
```

### Namespace Isolation for Micro-Frontends

```javascript
// Each MFE gets a namespaced view of the bus to prevent channel conflicts
class NamespacedBus {
  #bus;
  #namespace;

  constructor(bus, namespace) {
    this.#bus = bus;
    this.#namespace = namespace;
  }

  #prefixed(channel) {
    return `${this.#namespace}:${channel}`;
  }

  on(channel, handler, opts) {
    return this.#bus.on(this.#prefixed(channel), handler, opts);
  }

  publish(channel, data, meta) {
    return this.#bus.publish(this.#prefixed(channel), data, meta);
  }

  // Cross-namespace: subscribe to another MFE's events
  listenTo(namespace, channel, handler) {
    return this.#bus.on(`${namespace}:${channel}`, handler);
  }
}

// Navigation MFE
const navBus = new NamespacedBus(globalBus, "nav");
navBus.publish("route:changed", { to: "/settings" });
// Publishes as: 'nav:route:changed'

// Settings MFE listens to nav events
const settingsBus = new NamespacedBus(globalBus, "settings");
settingsBus.listenTo("nav", "route:changed", ({ to }) => {
  if (to === "/settings") loadSettings();
});
```

---

## 12. Request/Reply Pattern

Standard pub/sub is fire-and-forget. For request/reply, extend it with correlation IDs.

```javascript
class RequestReplyBus extends EventBus {
  /**
   * Send a request and wait for a reply.
   */
  async request(channel, data, timeoutMs = 5000) {
    const correlationId = Math.random().toString(36).slice(2);
    const replyChannel = `reply:${channel}:${correlationId}`;

    return new Promise((resolve, reject) => {
      const timeoutId = setTimeout(() => {
        unsub();
        reject(
          new Error(`Request timeout for "${channel}" after ${timeoutMs}ms`),
        );
      }, timeoutMs);

      // Listen for the reply on the correlation channel
      const unsub = this.once(replyChannel, (response) => {
        clearTimeout(timeoutId);
        if (response.error) {
          reject(new Error(response.error));
        } else {
          resolve(response.data);
        }
      });

      // Publish the request with the correlation ID
      this.publish(channel, data, { correlationId, replyChannel });
    });
  }

  /**
   * Register a service that handles requests on a channel.
   */
  reply(channel, handler) {
    return this.on(channel, async (data, meta) => {
      try {
        const result = await handler(data, meta);
        this.publish(meta.replyChannel, { data: result });
      } catch (err) {
        this.publish(meta.replyChannel, { error: err.message });
      }
    });
  }
}

// Usage: RPC-style communication between modules
const bus = new RequestReplyBus();

// Service: handles user lookup requests
bus.reply("service:user:get", async ({ userId }) => {
  const user = await userRepository.find(userId);
  if (!user) throw new Error(`User ${userId} not found`);
  return user;
});

// Client: requests user data
const user = await bus.request("service:user:get", { userId: "42" });
console.log("Got user:", user.name);
```

---

## 13. Dead Letter Queue

Events with no subscribers can be lost silently. A Dead Letter Queue captures them for inspection.

```javascript
class EventBusWithDLQ extends EventBus {
  #dlq = []; // dead letter queue
  #maxDLQSize;

  constructor(options = {}) {
    super(options);
    this.#maxDLQSize = options.dlqSize ?? 100;
  }

  publish(channel, data, meta = {}) {
    const notified = super.publish(channel, data, meta);

    if (notified === 0) {
      const entry = { channel, data, meta, timestamp: Date.now() };
      this.#dlq.push(entry);

      // Trim DLQ
      if (this.#dlq.length > this.#maxDLQSize) this.#dlq.shift();

      console.warn(`[EventBus] Dead letter: "${channel}" had no subscribers`);
    }

    return notified;
  }

  getDLQ() {
    return [...this.#dlq];
  }

  clearDLQ() {
    this.#dlq = [];
  }

  /**
   * Replay dead letter messages for a channel when a subscriber joins.
   */
  replayDLQ(channel, handler) {
    const messages = this.#dlq.filter((e) => e.channel === channel);
    messages.forEach(({ data, meta }) => {
      try {
        handler(data, meta);
      } catch (err) {
        console.error("DLQ replay error:", err);
      }
    });
    this.#dlq = this.#dlq.filter((e) => e.channel !== channel); // remove replayed
  }
}
```

---

## 14. Testing Pub/Sub Systems

Testing event-driven code requires being able to inspect what was published and when.

```javascript
class TestEventBus extends EventBus {
  #published = [];

  publish(channel, data, meta = {}) {
    this.#published.push({ channel, data, meta, timestamp: Date.now() });
    return super.publish(channel, data, meta);
  }

  // Test assertions
  wasPublished(channel, matcher = null) {
    return this.#published.some(
      (e) => e.channel === channel && (matcher === null || matcher(e.data)),
    );
  }

  publishedCount(channel) {
    return this.#published.filter((e) => e.channel === channel).length;
  }

  getPublished(channel) {
    return this.#published
      .filter((e) => e.channel === channel)
      .map((e) => e.data);
  }

  reset() {
    this.#published = [];
    this.clear();
  }
}

// Test usage (Jest/Vitest style)
describe("UserModule", () => {
  let bus;

  beforeEach(() => {
    bus = new TestEventBus();
    // Inject test bus into module
    userModule.setBus(bus);
  });

  it("publishes user:authenticated on successful login", async () => {
    await userModule.login({ email: "test@test.com", password: "pass" });

    expect(bus.wasPublished("user:authenticated")).toBe(true);
    expect(
      bus.wasPublished("user:authenticated", (d) => d.userId === "42"),
    ).toBe(true);
  });

  it("publishes bus:error if login fails", async () => {
    mockAuthApi.login.mockRejectedValue(new Error("Invalid credentials"));
    await userModule.login({ email: "bad", password: "bad" });

    expect(bus.wasPublished("bus:error")).toBe(true);
  });
});
```

---

## 15. Memory Management

Pub/Sub systems are a common source of memory leaks. The bus holds references to all subscribers — if subscribers close over large objects or component instances, those stay alive indefinitely.

### Automatic Subscriber Cleanup with WeakRef

```javascript
// Use WeakRef so the bus doesn't prevent subscriber objects from being GC'd
class WeakRefEventBus extends EventBus {
  onWeak(channel, targetObject, method) {
    const weakRef = new WeakRef(targetObject);

    const handler = (...args) => {
      const obj = weakRef.deref();
      if (obj) {
        obj[method](...args);
      } else {
        // Object was GC'd — auto-unsubscribe
        unsub();
      }
    };

    const unsub = this.on(channel, handler);
    return unsub;
  }
}

// Usage: subscriber auto-removed when component is GC'd
bus.onWeak("data:updated", dashboardComponent, "onDataUpdate");
// dashboardComponent = null → component GC'd → handler auto-removes itself
```

### Subscriber Limits and Leak Detection

```javascript
// Warn when a channel accumulates too many subscribers
const MAX_SUBSCRIBERS_PER_CHANNEL = 20;

bus.on = new Proxy(bus.on.bind(bus), {
  apply(target, thisArg, [channel, handler, options]) {
    const result = Reflect.apply(target, thisArg, [channel, handler, options]);
    const count = bus.stats()[channel] ?? 0;

    if (count > MAX_SUBSCRIBERS_PER_CHANNEL) {
      console.warn(
        `[EventBus] Warning: ${count} subscribers on "${channel}". ` +
          `Possible memory leak — are unsubscribes being called?`,
      );
    }

    return result;
  },
});
```

---

## 16. Good Practices

### ✅ Use namespaced channel names

```javascript
// ✅ Clear, collision-resistant channel names
// Format: domain:entity:action
bus.publish("user:profile:updated", data);
bus.publish("cart:item:added", data);
bus.publish("api:request:failed", data);
bus.publish("ui:modal:opened", data);

// ❌ Vague, collision-prone
bus.publish("updated", data);
bus.publish("change", data);
```

### ✅ Always store and call unsubscribe functions

```javascript
// ✅ Collect all unsubscribes, call them on cleanup
class Feature {
  #unsubscribes = [];

  init() {
    this.#unsubscribes.push(bus.on("user:login", this.#onLogin));
    this.#unsubscribes.push(bus.on("user:logout", this.#onLogout));
    this.#unsubscribes.push(bus.on("data:update", this.#onData));
  }

  destroy() {
    this.#unsubscribes.forEach((fn) => fn());
    this.#unsubscribes = [];
  }
}
```

### ✅ Use `once` for initialization events

```javascript
// ✅ Auto-unsubscribes after first message — no manual cleanup needed
bus.once("app:ready", () => {
  initializeDashboard();
  // handler automatically removed after this fires
});
```

### ✅ Document channel contracts

```javascript
/**
 * Channel: 'user:authenticated'
 * Published by: authModule
 * Payload: {
 *   userId: string,
 *   role: 'admin' | 'user' | 'guest',
 *   timestamp: number,
 * }
 * Subscribers: analyticsModule, permissionsModule, navigationModule
 */
bus.publish("user:authenticated", { userId, role, timestamp });
```

---

## 17. Bad Practices

### ❌ Chaining everything through one global bus

```javascript
// ❌ Everything through one bus — no isolation, debugging nightmare
bus.publish("click", { x: 100, y: 200 }); // mouse events?
bus.publish("response", { data: apiData }); // API responses?
bus.publish("resize", { width: 800 }); // resize events?
bus.publish("done", { result }); // what's "done"?
```

### ❌ Subscriptions without cleanup

```javascript
// ❌ Memory leak — subscriptions accumulate on each render/mount
function renderComponent(data) {
  bus.on("theme:change", updateStyles); // new subscription every call!
  // After 100 renders: 100 subscriptions for the same handler
}
```

### ❌ Complex logic inside subscribe callbacks

```javascript
// ❌ Business logic in event handler — hard to test, hard to maintain
bus.on("order:created", async (order) => {
  // 100 lines of order processing logic here
  // Can't be tested without triggering the bus
  // Can't be reused outside of event context
});

// ✅ Call service functions — testable independently
bus.on("order:created", (order) => orderService.process(order));
```

### ❌ Synchronous heavy work in pub/sub callbacks

```javascript
// ❌ Heavy sync work blocks all subsequent subscribers
bus.on("data:received", (data) => {
  const processed = heavySyncProcessing(data); // 200ms — blocks everything
  updateUI(processed);
});

// ✅ Defer heavy work
bus.on("data:received", (data) => {
  requestIdleCallback(() => {
    const processed = heavySyncProcessing(data);
    updateUI(processed);
  });
});
```

---

## 18. Common Mistakes

### Mistake 1 — Subscribing to channels before the bus is initialized

```javascript
// ❌ Race condition — bus may not exist yet
bus.on("app:ready", initModule); // bus is undefined if module loads first

// ✅ Ensure bus is initialized before module subscribes
// Use module initialization order, dependency injection, or deferred setup
```

### Mistake 2 — Publishing in subscriber causing infinite loop

```javascript
// ❌ Infinite loop: 'data:updated' → 'data:processed' → 'data:updated'
bus.on("data:updated", (data) => {
  const processed = transform(data);
  bus.publish("data:processed", processed);
});

bus.on("data:processed", (data) => {
  const normalized = normalize(data);
  bus.publish("data:updated", normalized); // loops back!
});
```

### Mistake 3 — Expecting synchronous notification order across modules

```javascript
// Subscribers are called in subscription order (or priority)
// Don't assume which module subscribed first in a large codebase

// ❌ Relying on ordering:
bus.publish("state:changed", newState);
// Assuming module A handles it before module B uses the result
// This is fragile — use explicit sequencing or request/reply
```

### Mistake 4 — Sharing mutable data in events

```javascript
// ❌ Mutable shared data — subscriber can accidentally mutate it
bus.publish("cart:updated", cart); // direct reference to cart object

bus.on("cart:updated", (cart) => {
  cart.items.push(newItem); // mutates the original cart object!
  // Other subscribers now see different cart
});

// ✅ Publish immutable copies
bus.publish("cart:updated", Object.freeze(structuredClone(cart)));
```

---

## 19. Interview-Level Explanation

> **"What is Pub/Sub? How would you implement an event bus? How do you use it in large-scale frontend apps?"**

**Strong answer:**

> "Pub/Sub is a messaging pattern where publishers emit events to a central broker — the event bus — and subscribers register interest in specific event types, with neither side knowing about the other. It's a more decoupled variant of the Observer pattern: in Observer, the subject holds direct references to observers; in Pub/Sub, an intermediary bus handles routing.
>
> A minimal event bus needs four things: a channel registry (typically a Map from channel name to a Set of handlers), a `publish` method that iterates the Set and calls each handler, a `subscribe` method that adds a handler and returns an unsubscribe function, and error isolation so one failing subscriber doesn't break others.
>
> For production systems, I'd add: typed channels (validate payloads at dev time), middleware for logging and rate limiting, message history for late subscribers, priority ordering for cases like auth guards needing to run before UI handlers, and dead letter tracking for published events with no subscribers.
>
> In large frontend apps, Pub/Sub is the foundation for cross-module communication. User authentication, analytics, feature flag changes, navigation events — these are all good pub/sub candidates because they affect many modules but none should have a direct dependency on the others. In micro-frontend architectures, a shared global event bus is often exposed on `window` so independently deployed apps can coordinate without shared code.
>
> The biggest risk is memory leaks: subscribers that are never removed keep their closures alive. The pattern is: every `subscribe` call returns an unsubscribe function, and every component that subscribes must call those functions in its cleanup/unmount lifecycle. I'd also add subscriber count warnings per channel to detect leak patterns in development."

---

## 20. Exercises

### Exercise 1 — Implement a complete EventBus from scratch

Implement an EventBus class with:

- `on(channel, handler)` → returns unsubscribe fn
- `once(channel, handler)` → auto-unsubscribes after first event
- `emit(channel, ...args)` → calls all handlers, returns count
- Error isolation between handlers
- Warning when a channel exceeds 10 subscribers

<details>
<summary>Solution</summary>

```javascript
class EventBus {
  #channels = new Map();
  #maxListeners = 10;

  on(channel, handler) {
    if (typeof handler !== "function")
      throw new TypeError("Handler must be a function");
    if (!this.#channels.has(channel)) this.#channels.set(channel, new Set());

    const handlers = this.#channels.get(channel);
    handlers.add(handler);

    if (handlers.size > this.#maxListeners) {
      console.warn(
        `[EventBus] ${handlers.size} listeners on "${channel}" — possible leak`,
      );
    }

    return () => {
      handlers.delete(handler);
      if (handlers.size === 0) this.#channels.delete(channel);
    };
  }

  once(channel, handler) {
    const unsub = this.on(channel, function onceHandler(...args) {
      handler(...args);
      unsub();
    });
    return unsub;
  }

  emit(channel, ...args) {
    const handlers = this.#channels.get(channel);
    if (!handlers || handlers.size === 0) return 0;

    let count = 0;
    for (const handler of [...handlers]) {
      try {
        handler(...args);
        count++;
      } catch (err) {
        console.error(`[EventBus] Error in handler for "${channel}":`, err);
      }
    }
    return count;
  }

  off(channel, handler) {
    this.#channels.get(channel)?.delete(handler);
  }

  clear(channel) {
    if (channel) this.#channels.delete(channel);
    else this.#channels.clear();
  }
}
```

</details>

---

### Exercise 2 — Build a micro-frontend coordinator

Using the EventBus from Exercise 1, coordinate three independent modules:

- `AuthModule`: publishes `auth:login` and `auth:logout`
- `CartModule`: subscribes to `auth:logout` to clear the cart
- `NavigationModule`: subscribes to `auth:login` to redirect to dashboard

```javascript
// Implement these three modules communicating ONLY through the event bus
// No direct imports between modules
```

<details>
<summary>Solution</summary>

```javascript
const bus = new EventBus();

// auth-module.js
const AuthModule = {
  async login(credentials) {
    const user = await fakeAuthAPI(credentials);
    bus.emit("auth:login", { userId: user.id, role: user.role });
    return user;
  },
  logout() {
    bus.emit("auth:logout", { timestamp: Date.now() });
  },
};

// cart-module.js
const CartModule = {
  items: [],
  init() {
    bus.on("auth:logout", () => {
      this.items = [];
      console.log("[Cart] Cleared on logout");
    });
  },
};

// navigation-module.js
const NavigationModule = {
  init() {
    bus.on("auth:login", ({ role }) => {
      const dest = role === "admin" ? "/admin" : "/dashboard";
      console.log(`[Nav] Redirecting to ${dest}`);
      // window.location.href = dest;
    });
  },
};

// Bootstrap
CartModule.init();
NavigationModule.init();

// Test
await AuthModule.login({ email: "admin@test.com", password: "123" });
// [Nav] Redirecting to /admin

AuthModule.logout();
// [Cart] Cleared on logout
```

</details>

---

### Exercise 3 — Add middleware to log all events in development

Extend the EventBus with a middleware system and add:

1. A development logger that logs every event with timestamp
2. A performance tracer that measures handler execution time
3. A filter that blocks any event starting with `internal:`

<details>
<summary>Solution</summary>

```javascript
// Development logger middleware
const devLogger = (event) => {
  if (process.env.NODE_ENV === "development") {
    console.log(
      `%c[EventBus] ${event.channel}`,
      "color: #4CAF50; font-weight: bold",
      event.data,
    );
  }
  return event; // pass through
};

// Performance tracer middleware
const perfTracer = (event) => {
  const start = performance.now();
  const wrapped = {
    ...event,
    meta: {
      ...event.meta,
      _traceStart: start,
    },
  };
  return wrapped;
};

// Security filter middleware
const securityFilter = (event) => {
  if (event.channel.startsWith("internal:")) {
    console.warn(`[EventBus] Blocked internal channel: "${event.channel}"`);
    return null; // block
  }
  return event;
};

const bus = new EventBus({ debug: false });
bus.use(devLogger);
bus.use(perfTracer);
bus.use(securityFilter);

bus.publish("user:login", { userId: "1" }); // ✅ logged + traced
bus.publish("internal:secret", { key: "val" }); // ❌ blocked
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/14-observer-patterns.md`](./14-observer-patterns.md) — Observer vs Pub/Sub comparison
- [`system-design/06-event-driven-frontend.md`](../system-design/06-event-driven-frontend.md) — Event-driven architecture at scale
- [`system-design/03-micro-frontends.md`](../system-design/03-micro-frontends.md) — Cross-MFE communication
- [`system-design/07-plugin-systems.md`](../system-design/07-plugin-systems.md) — Plugin architecture using events
- [`patterns/02-mediator.md`](../patterns/02-mediator.md) — Mediator pattern (event bus variant)

---

<div align="center">

**`javascript-core/` complete!** 🎉

All 15 files in the JavaScript Core section are done.

**Next section:** [`browser-internals/`](../browser-internals/) →

</div>
