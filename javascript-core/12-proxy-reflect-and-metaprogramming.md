# 27 — Proxy, Reflect, and Metaprogramming

> **"Metaprogramming is writing code that treats other code as data — intercepting property access, trapping function calls, virtualizing object behavior. Proxy and Reflect are the two tools JavaScript provides for this. They power Vue 3's reactivity, mock libraries, validation frameworks, and ORMs. Understanding them means understanding how those tools work at the lowest level they can."**

🔴 **Level: Senior**

---

## 📚 Table of Contents

1. [What Is Metaprogramming?](#1-what-is-metaprogramming)
2. [Proxy — The Interception Layer](#2-proxy--the-interception-layer)
3. [All 13 Proxy Traps](#3-all-13-proxy-traps)
4. [Reflect — The Companion API](#4-reflect--the-companion-api)
5. [Why Proxy + Reflect Belong Together](#5-why-proxy--reflect-belong-together)
6. [Reactive Objects (Vue 3 Reactivity Model)](#6-reactive-objects-vue-3-reactivity-model)
7. [Validation with Proxy](#7-validation-with-proxy)
8. [Deep Proxy (Nested Reactivity)](#8-deep-proxy-nested-reactivity)
9. [Revocable Proxies](#9-revocable-proxies)
10. [Performance Considerations](#10-performance-considerations)
11. [Common Mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)

---

## 1. What Is Metaprogramming?

```javascript
// NORMAL programming: writing code that operates on DATA
const users = fetchUsers();
users.filter((u) => u.active);

// METAPROGRAMMING: writing code that operates on other CODE or its BEHAVIOR
// Examples:
//   - Intercepting property access to track what fields were read
//   - Automatically validating values when a property is set
//   - Making a non-existent property appear to exist (virtual properties)
//   - Observing function calls for mocking/testing

// JavaScript's metaprogramming primitives:
// 1. Proxy     — intercept fundamental object operations
// 2. Reflect   — invoke the default behavior of those operations
// 3. Symbol    — hook into built-in protocols (Symbol.iterator, Symbol.toPrimitive, etc.)
// 4. Object.defineProperty — control property descriptor attributes
```

---

## 2. Proxy — The Interception Layer

```javascript
// Proxy wraps a target object and intercepts operations via a handler
const proxy = new Proxy(target, handler);

// Minimal example — log every property access
const user = { name: "Alice", age: 30 };

const loggedUser = new Proxy(user, {
  get(target, prop, receiver) {
    console.log(`GET ${String(prop)}`);
    return Reflect.get(target, prop, receiver); // default behavior
  },
  set(target, prop, value, receiver) {
    console.log(`SET ${String(prop)} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  },
});

loggedUser.name; // logs "GET name", returns 'Alice'
loggedUser.age = 31; // logs "SET age = 31"

// The proxy IS the object from the caller's perspective:
loggedUser.name === user.name; // true — same underlying value
loggedUser instanceof Object; // true

// TRAP PARAMETERS follow a consistent signature:
// target   — the original object being proxied
// prop     — the property key (string or Symbol)
// receiver — the proxy itself (or the object that inherited from the proxy)
// value    — (for set) the new value being assigned
```

---

## 3. All 13 Proxy Traps

```javascript
const handler = {
  // Property access: obj.prop or obj[prop]
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);
  },

  // Property assignment: obj.prop = value
  set(target, prop, value, receiver) {
    return Reflect.set(target, prop, value, receiver); // must return true or throw
  },

  // Property existence: prop in obj
  has(target, prop) {
    return Reflect.has(target, prop);
  },

  // Property deletion: delete obj.prop
  deleteProperty(target, prop) {
    return Reflect.deleteProperty(target, prop); // must return true or throw
  },

  // Object.keys(), for...in, spread
  ownKeys(target) {
    return Reflect.ownKeys(target);
  },

  // Object.getOwnPropertyDescriptor(obj, prop)
  getOwnPropertyDescriptor(target, prop) {
    return Reflect.getOwnPropertyDescriptor(target, prop);
  },

  // Object.defineProperty(obj, prop, descriptor)
  defineProperty(target, prop, descriptor) {
    return Reflect.defineProperty(target, prop, descriptor);
  },

  // Object.getPrototypeOf(obj)
  getPrototypeOf(target) {
    return Reflect.getPrototypeOf(target);
  },

  // Object.setPrototypeOf(obj, proto)
  setPrototypeOf(target, proto) {
    return Reflect.setPrototypeOf(target, proto);
  },

  // Object.isExtensible(obj)
  isExtensible(target) {
    return Reflect.isExtensible(target);
  },

  // Object.preventExtensions(obj)
  preventExtensions(target) {
    return Reflect.preventExtensions(target);
  },

  // new obj() — only for function targets
  construct(target, args, newTarget) {
    return Reflect.construct(target, args, newTarget);
  },

  // obj() — only for function targets
  apply(target, thisArg, args) {
    return Reflect.apply(target, thisArg, args);
  },
};
```

---

## 4. Reflect — The Companion API

```javascript
// Reflect mirrors every Proxy trap as a function
// It provides the DEFAULT behavior for each operation

// Why Reflect.get instead of target[prop]?
// 1. Correctly forwards the `receiver` (important for inherited getters)
// 2. Consistent API — every trap has an exact Reflect counterpart
// 3. Returns false instead of throwing for operations like deleteProperty

// receiver matters for getters:
const parent = {
  get value() { return this._value; }  // `this` depends on receiver
};
const child = Object.create(parent);
child._value = 42;

// Without receiver: parent's getter runs with parent as `this` → this._value = undefined
Reflect.get(parent, 'value');                 // undefined (this = parent)

// With receiver: parent's getter runs with child as `this` → this._value = 42
Reflect.get(parent, 'value', child);          // 42 ✅

// Reflect vs traditional equivalents:
Reflect.get(obj, 'x')              ←→  obj.x
Reflect.set(obj, 'x', 1)          ←→  obj.x = 1       (returns boolean)
Reflect.has(obj, 'x')             ←→  'x' in obj       (returns boolean)
Reflect.deleteProperty(obj, 'x')  ←→  delete obj.x     (returns boolean)
Reflect.ownKeys(obj)              ←→  Object.getOwnPropertyNames(obj)
                                        + Object.getOwnPropertySymbols(obj)
Reflect.apply(fn, thisArg, args)  ←→  fn.apply(thisArg, args)
Reflect.construct(Cls, args)      ←→  new Cls(...args)
```

---

## 5. Why Proxy + Reflect Belong Together

```javascript
// A Proxy without Reflect in the get trap has a subtle bug:
const buggy = new Proxy(parent, {
  get(target, prop) {
    return target[prop]; // ❌ `this` inside any getter will be `target`, not the proxy
  },
});

// A Proxy with Reflect correctly forwards the receiver:
const correct = new Proxy(parent, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver); // ✅ receiver (the proxy) is forwarded
  },
});

// This matters when:
// - The target has getter functions that use `this`
// - Child objects inherit from a proxied parent
// - Any method on the proxied object calls other methods (this.method())

// RULE: in every Proxy trap, always use the corresponding Reflect method
// to invoke the default behavior, passing all arguments including receiver.
```

---

## 6. Reactive Objects (Vue 3 Reactivity Model)

A simplified version of Vue 3's `reactive()`:

```javascript
let activeEffect = null; // the currently-running reaction

function effect(fn) {
  activeEffect = fn;
  fn(); // run once to collect dependencies
  activeEffect = null;
}

function reactive(obj) {
  const deps = new Map(); // prop → Set<effect>

  function getDeps(prop) {
    if (!deps.has(prop)) deps.set(prop, new Set());
    return deps.get(prop);
  }

  return new Proxy(obj, {
    get(target, prop, receiver) {
      // TRACK: record which effect is reading this property
      if (activeEffect) {
        getDeps(prop).add(activeEffect);
      }
      return Reflect.get(target, prop, receiver);
    },

    set(target, prop, value, receiver) {
      const result = Reflect.set(target, prop, value, receiver);
      // TRIGGER: re-run all effects that read this property
      getDeps(prop).forEach((fn) => fn());
      return result;
    },
  });
}

// Usage:
const state = reactive({ count: 0, name: "Alice" });

effect(() => {
  console.log(`Count is: ${state.count}`); // runs now, and whenever count changes
});
// Logs: "Count is: 0"

state.count = 1; // Logs: "Count is: 1"
state.count = 2; // Logs: "Count is: 2"
state.name = "Bob"; // Nothing logged — name wasn't read in the effect
```

**Key insight:** this is structurally identical to how Vue 3's reactivity works. The `get` trap subscribes effects to properties; the `set` trap notifies them. The entire reactive system is built on two Proxy traps and a Map of Sets.

---

## 7. Validation with Proxy

```javascript
function createValidated(schema) {
  const data = {};

  return new Proxy(data, {
    set(target, prop, value) {
      const rule = schema[prop];
      if (!rule) throw new Error(`Unknown property: ${String(prop)}`);

      const error = rule.validate(value);
      if (error) throw new TypeError(`${String(prop)}: ${error}`);

      return Reflect.set(target, prop, value);
    },

    get(target, prop, receiver) {
      if (
        !(prop in target) &&
        prop in schema &&
        schema[prop].default !== undefined
      ) {
        return schema[prop].default;
      }
      return Reflect.get(target, prop, receiver);
    },
  });
}

const user = createValidated({
  name: {
    validate: (v) => (typeof v !== "string" ? "Must be a string" : null),
    default: "Anonymous",
  },
  age: {
    validate: (v) =>
      typeof v !== "number" || v < 0 || v > 150
        ? "Must be a number between 0 and 150"
        : null,
  },
});

user.name = "Alice"; // ✅
user.age = 30; // ✅
user.age = -1; // ❌ TypeError: age: Must be a number between 0 and 150
user.email = "x"; // ❌ Error: Unknown property: email
user.name; // 'Alice'
// If name was never set: 'Anonymous' (default)
```

---

## 8. Deep Proxy (Nested Reactivity)

```javascript
// A flat Proxy only intercepts operations on the top-level object.
// For nested objects, you need to wrap them lazily on access:

function deepReactive(obj) {
  if (typeof obj !== "object" || obj === null) return obj;

  return new Proxy(obj, {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      // Lazily wrap nested objects when they are accessed
      if (typeof value === "object" && value !== null) {
        return deepReactive(value); // recursive — returns a new Proxy each time
      }
      return value;
    },

    set(target, prop, value, receiver) {
      console.log(`Set ${String(prop)} =`, value);
      return Reflect.set(target, prop, value, receiver);
    },
  });
}

const state = deepReactive({ user: { address: { city: "NY" } } });
state.user.address.city; // triggers 3 get traps (user, address, city)
state.user.address.city = "LA"; // triggers 3 get traps + 1 set trap on city

// ⚠️ Performance note: a new Proxy is created on EVERY nested access.
// Vue 3 caches the proxied version: the first time obj.nested is accessed,
// the Proxy is stored in a WeakMap so subsequent accesses return the same Proxy.
const proxyCache = new WeakMap();
function deepReactiveCached(obj) {
  if (typeof obj !== "object" || obj === null) return obj;
  if (proxyCache.has(obj)) return proxyCache.get(obj);
  const proxy = new Proxy(obj, {
    /* ... */
  });
  proxyCache.set(obj, proxy);
  return proxy;
}
```

---

## 9. Revocable Proxies

```javascript
// Proxy.revocable: a proxy that can be permanently disabled
const { proxy, revoke } = Proxy.revocable(target, handler);

proxy.name; // works normally

revoke(); // permanently disable the proxy — CANNOT be undone

proxy.name; // ❌ TypeError: Cannot perform 'get' on a proxy that has been revoked

// USE CASES:
// 1. Time-limited access: provide a proxy for a session, revoke when session ends
// 2. Capability-based security: hand out a proxy, revoke to withdraw access
// 3. Cleanup: ensure no references to internal data survive beyond a lifecycle

function createSessionProxy(resource) {
  const { proxy, revoke } = Proxy.revocable(resource, {
    get(target, prop, receiver) {
      return Reflect.get(target, prop, receiver);
    },
  });

  // Automatically revoke after 5 minutes
  setTimeout(revoke, 5 * 60 * 1000);

  return { proxy, revoke };
}
```

---

## 10. Performance Considerations

```javascript
// Proxy traps add overhead to EVERY intercepted operation.
// In hot paths (tight loops, frequent property access), this can be measurable.

// Benchmark: property access on a plain object vs a Proxy
const plain = { x: 1 };
const proxied = new Proxy({ x: 1 }, { get: (t, p, r) => Reflect.get(t, p, r) });

// Plain: ~1-2ns per access
// Proxy: ~10-50ns per access (5-25x slower in V8 as of 2024)

// GUIDELINES:
// ✅ Proxy is fine for:
//   - Configuration objects accessed a handful of times
//   - Reactive state management where re-renders are the actual bottleneck
//   - API boundaries, validation, mocking

// ⚠️ Avoid Proxy in:
//   - Tight loops reading/writing thousands of properties per frame
//   - Core data structures accessed millions of times per second
//   - WebAssembly / TypedArray hot paths

// Profiling: Proxy overhead only appears in the "get" / "set" operations
// in a CPU profile — look for surprisingly hot property reads on hot objects
```

---

## 11. Common Mistakes

### Mistake 1 — set trap must return true

```javascript
// ❌ Missing return — returns undefined (falsy) → TypeError in strict mode
const p = new Proxy({}, {
  set(target, prop, value) {
    target[prop] = value;
    // no return!
  }
});
p.x = 1; // TypeError: 'set' on proxy: trap returned falsish

// ✅ Always return the result of Reflect.set (or true explicitly)
set(target, prop, value, receiver) {
  return Reflect.set(target, prop, value, receiver);
}
```

### Mistake 2 — Invariant violations (sealed/frozen targets)

```javascript
// Proxy traps must respect the target object's invariants
const frozen = Object.freeze({ x: 1 });
const p = new Proxy(frozen, {
  get(target, prop) {
    return 99; // ❌ TypeError: cannot report different value for non-writable property
  },
});
p.x; // throws — trap lies about a non-writable, non-configurable property
```

### Mistake 3 — Proxying a class method loses `this`

```javascript
class Counter {
  #n = 0;
  inc() {
    this.#n++;
  }
  get value() {
    return this.#n;
  }
}
const c = new Counter();
const p = new Proxy(c, {});
p.inc(); // ❌ TypeError — `this` inside inc() is the Proxy, not the Counter instance
// Private fields (#n) are only accessible when `this` is the exact target

// ✅ Bind methods to the target in the get trap
const p = new Proxy(c, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    return typeof value === "function" ? value.bind(target) : value;
  },
});
p.inc(); // ✅ works — this = counter instance
```

---

## 12. Exercises

### Exercise 1 — Implement a simple observable

```javascript
// Implement observable(obj) that returns a proxy where calling
// obj.onChange(callback) registers a listener, and any set operation
// calls all listeners with (property, newValue, oldValue).
```

<details>
<summary>Solution</summary>

```javascript
function observable(obj) {
  const listeners = new Set();

  return new Proxy(obj, {
    get(target, prop, receiver) {
      if (prop === "onChange") {
        return (cb) => listeners.add(cb);
      }
      return Reflect.get(target, prop, receiver);
    },

    set(target, prop, value, receiver) {
      const oldValue = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      listeners.forEach((cb) => cb(prop, value, oldValue));
      return result;
    },
  });
}

const state = observable({ count: 0 });
state.onChange((prop, newVal, oldVal) => {
  console.log(`${prop}: ${oldVal} → ${newVal}`);
});
state.count = 1; // "count: 0 → 1"
state.count = 2; // "count: 1 → 2"
```

</details>

---

## 🔗 Related Topics

- [`24-es6-modern-syntax.md`](./24-es6-modern-syntax.md) — Symbols (Symbol.toPrimitive, Symbol.iterator)
- [`21-objects-and-destructuring.md`](./21-objects-and-destructuring.md) — Property descriptors
- [`09-garbage-collection.md`](./09-garbage-collection.md) — WeakMap for proxy caches
