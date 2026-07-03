# 21 — Objects and Destructuring

> **"An object is a bag of properties. That's it. Everything else in JavaScript — arrays, functions, dates, DOM elements — is also an object. Understanding how to create, clone, merge, and inspect objects, and how destructuring makes their consumption ergonomic, covers a huge fraction of everyday JavaScript code."**

🟢 **Level: Beginner → Intermediate**

---

## 📚 Table of Contents

1. [Creating Objects](#1-creating-objects)
2. [Reading and Writing Properties](#2-reading-and-writing-properties)
3. [Computed Property Names](#3-computed-property-names)
4. [Shorthand Property and Method Syntax](#4-shorthand-property-and-method-syntax)
5. [Object Destructuring](#5-object-destructuring)
6. [Spread and Rest with Objects](#6-spread-and-rest-with-objects)
7. [Object Methods Reference](#7-object-methods-reference)
8. [Property Descriptors](#8-property-descriptors)
9. [Cloning and Merging Objects](#9-cloning-and-merging-objects)
10. [Common Patterns](#10-common-patterns)
11. [Common Mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)

---

## 1. Creating Objects

```javascript
// Object literal (preferred)
const person = {
  name: "Alice",
  age: 30,
  greet() {
    return `Hi, I'm ${this.name}`;
  },
};

// Constructor function (pre-class)
function Person(name, age) {
  this.name = name;
  this.age = age;
}
const alice = new Person("Alice", 30);

// Class syntax (ES6 — syntactic sugar over constructor functions)
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  greet() {
    return `Hi, I'm ${this.name}`;
  }
}
const alice = new Person("Alice", 30);

// Object.create (explicit prototype)
const animal = {
  breathe() {
    return "breathing";
  },
};
const dog = Object.create(animal);
dog.bark = function () {
  return "woof";
};
dog.breathe(); // found on prototype

// Factory function (returns plain objects)
function createPerson(name, age) {
  return {
    name,
    age,
    greet() {
      return `Hi, I'm ${this.name}`;
    },
  };
}
const bob = createPerson("Bob", 25);
```

---

## 2. Reading and Writing Properties

```javascript
const user = { name: "Alice", age: 30, "full name": "Alice Smith" };

// Dot notation (when key is a valid identifier)
user.name; // 'Alice'
user.age; // 30

// Bracket notation (always works, required for special characters)
user["name"]; // 'Alice'
user["full name"]; // 'Alice Smith'

// Dynamic property access
const key = "age";
user[key]; // 30

// Writing
user.email = "alice@example.com"; // add new property
user.age = 31; // update existing

// Deleting
delete user.age;
"age" in user; // false

// Checking existence
"name" in user; // true (checks own AND inherited properties)
user.hasOwnProperty("name"); // true (own properties only)
Object.hasOwn(user, "name"); // true (modern version, ES2022)

// Optional chaining with objects
const street = user?.address?.street; // undefined if address doesn't exist
```

---

## 3. Computed Property Names

```javascript
// Dynamic key from an expression
const prefix = "user";
const obj = {
  [`${prefix}Name`]: "Alice", // key = 'userName'
  [`${prefix}Age`]: 30, // key = 'userAge'
};

// With variables
const id = "abc123";
const data = {
  [id]: { name: "Alice" }, // key = 'abc123'
};

// Symbol as key (unique, hidden from most iteration)
const SECRET = Symbol("secret");
const obj2 = {
  [SECRET]: "private value",
  public: "visible",
};
obj2[SECRET]; // 'private value'
Object.keys(obj2); // ['public'] — Symbol keys not included
```

---

## 4. Shorthand Property and Method Syntax

```javascript
const name = "Alice";
const age = 30;

// ❌ Old verbose way
const user = { name: name, age: age };

// ✅ Shorthand property (key name matches variable name)
const user = { name, age }; // { name: 'Alice', age: 30 }

// ❌ Old verbose method
const obj = {
  greet: function () {
    return "Hello";
  },
};

// ✅ Shorthand method
const obj = {
  greet() {
    return "Hello";
  }, // regular method
  async fetchData() {
    /* ... */
  }, // async method
  get name() {
    return this._name;
  }, // getter
  set name(v) {
    this._name = v;
  }, // setter
};
```

---

## 5. Object Destructuring

```javascript
const user = { name: "Alice", age: 30, city: "NY", role: "admin" };

// Basic destructuring
const { name, age } = user;
// name = 'Alice', age = 30

// Renaming while destructuring
const { name: userName, age: userAge } = user;
// userName = 'Alice', userAge = 30

// Default values
const { country = "USA", name: personName } = user;
// country = 'USA' (default, not in object), personName = 'Alice'

// Nested destructuring
const config = { db: { host: "localhost", port: 5432 }, debug: true };
const {
  db: { host, port },
  debug,
} = config;
// host = 'localhost', port = 5432, debug = true

// Rest in destructuring
const { name: n, age: a, ...rest } = user;
// n = 'Alice', a = 30, rest = { city: 'NY', role: 'admin' }

// In function parameters
function display({ name, age, city = "Unknown" }) {
  return `${name}, ${age}, ${city}`;
}
display(user); // 'Alice, 30, NY'
display({ name: "Bob", age: 25 }); // 'Bob, 25, Unknown'

// From a function return value
function getUser() {
  return { id: 1, name: "Alice", isAdmin: false };
}
const { id, name: uName } = getUser();
```

---

## 6. Spread and Rest with Objects

```javascript
// Spread: copy properties from one object to another (shallow)
const defaults = { theme: "light", lang: "en", debug: false };
const userPrefs = { theme: "dark", lang: "fr" };

// Merge (later values override earlier)
const merged = { ...defaults, ...userPrefs };
// { theme: 'dark', lang: 'fr', debug: false }

// Shallow copy
const copy = { ...user };
// Changing copy.name doesn't affect user.name (primitive)
// BUT: copy.address.city = 'LA' DOES affect user.address.city (reference)

// Add / override individual properties
const updated = { ...user, age: 31, email: "new@email.com" };
// user is unchanged, updated has new age and email

// Remove a property (via rest)
const { password, ...safeUser } = {
  name: "Alice",
  password: "secret",
  age: 30,
};
// safeUser = { name: 'Alice', age: 30 } — password excluded
```

---

## 7. Object Methods Reference

```javascript
const obj = { a: 1, b: 2, c: 3 };

// Keys, values, entries
Object.keys(obj); // ['a', 'b', 'c']
Object.values(obj); // [1, 2, 3]
Object.entries(obj); // [['a', 1], ['b', 2], ['c', 3]]

// Convert entries back to an object
Object.fromEntries([
  ["a", 1],
  ["b", 2],
]); // { a: 1, b: 2 }

// Practical: transform object values
const doubled = Object.fromEntries(
  Object.entries(obj).map(([key, value]) => [key, value * 2]),
);
// { a: 2, b: 4, c: 6 }

// Assign (shallow merge, mutates target)
const target = { a: 1 };
Object.assign(target, { b: 2 }, { c: 3 }); // target = { a:1, b:2, c:3 }
const shallowCopy = Object.assign({}, obj); // copy without mutation

// Freeze and seal
const frozen = Object.freeze({ x: 1 });
frozen.x = 99; // silently fails (or TypeError in strict mode)
frozen.y = 99; // silently fails
Object.isFrozen(frozen); // true

const sealed = Object.seal({ x: 1 });
sealed.x = 99; // ✅ allowed — can update existing properties
sealed.y = 99; // silently fails — can't ADD new properties
Object.isSealed(sealed); // true

// Define multiple properties at once
const user = {};
Object.defineProperties(user, {
  id: { value: 1, enumerable: true, writable: false },
  name: { value: "Alice", enumerable: true, writable: true },
});
```

---

## 8. Property Descriptors

```javascript
// Every property has a descriptor with 4 attributes:
// value, writable, enumerable, configurable

// Read descriptor
Object.getOwnPropertyDescriptor({ x: 1 }, "x");
// { value: 1, writable: true, enumerable: true, configurable: true }

// Define with custom descriptor
const obj = {};
Object.defineProperty(obj, "id", {
  value: 42,
  writable: false, // cannot be reassigned
  enumerable: false, // hidden from for...in, Object.keys(), JSON.stringify()
  configurable: false, // cannot be deleted or re-defined
});

obj.id = 99; // silently fails (strict: TypeError)
Object.keys(obj); // [] — 'id' is non-enumerable
JSON.stringify(obj); // '{}' — non-enumerable properties excluded

// Getters and setters via defineProperty
const circle = { radius: 5 };
Object.defineProperty(circle, "area", {
  get() {
    return Math.PI * this.radius ** 2;
  },
  enumerable: true,
  configurable: true,
});
circle.area; // ~78.54 — computed on access, no stored value
```

---

## 9. Cloning and Merging Objects

```javascript
// Shallow clone (3 approaches — all equivalent for plain objects)
const clone1 = { ...original };
const clone2 = Object.assign({}, original);
const clone3 = Object.fromEntries(Object.entries(original));

// Deep clone
const deepClone = structuredClone(original); // modern (Node 17+, Chrome 98+)
// Handles: nested objects, arrays, Date, Map, Set, circular references
// Does NOT handle: functions, symbols, class instances

// Manual deep clone (for full control):
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== "object") return obj;
  if (seen.has(obj)) return seen.get(obj);
  if (obj instanceof Date) return new Date(obj);
  if (Array.isArray(obj)) {
    const arr = [];
    seen.set(obj, arr);
    obj.forEach((item, i) => {
      arr[i] = deepClone(item, seen);
    });
    return arr;
  }
  const copy = {};
  seen.set(obj, copy);
  Object.keys(obj).forEach((key) => {
    copy[key] = deepClone(obj[key], seen);
  });
  return copy;
}
```

---

## 10. Common Patterns

```javascript
// Option object / config parameter pattern
function request(
  url,
  { method = "GET", headers = {}, body = null, timeout = 5000 } = {},
) {
  // Each option has a sensible default via destructuring
}
request("/api/users"); // all defaults
request("/api/users", { method: "POST", body: JSON.stringify(data) });

// Builder pattern via method chaining
class QueryBuilder {
  #query = { select: "*", from: "", where: [], limit: null };
  select(fields) {
    this.#query.select = fields;
    return this;
  }
  from(table) {
    this.#query.from = table;
    return this;
  }
  where(cond) {
    this.#query.where.push(cond);
    return this;
  }
  limit(n) {
    this.#query.limit = n;
    return this;
  }
  build() {
    return this.#query;
  }
}
const q = new QueryBuilder().from("users").where("active=1").limit(10).build();

// Namespace / module pattern
const UserService = {
  async getUser(id) {
    return api.get(`/users/${id}`);
  },
  async updateUser(id, data) {
    return api.put(`/users/${id}`, data);
  },
  async deleteUser(id) {
    return api.delete(`/users/${id}`);
  },
};
```

---

## 11. Common Mistakes

### Mistake 1 — Mutating a shared object reference

```javascript
const defaults = { theme: "light", debug: false };

// ❌ Mutates the shared defaults object
function configure(options) {
  options.debug = true; // if options === defaults, you've changed defaults!
  return options;
}

// ✅ Merge into a new object
function configure(options) {
  return { ...defaults, ...options, debug: true };
}
```

### Mistake 2 — Destructuring a null/undefined value

```javascript
// ❌ TypeError: Cannot destructure property 'name' of undefined
const { name } = fetchUser(); // if fetchUser() returns undefined/null

// ✅ Default the entire value
const { name } = fetchUser() ?? {};
const { name } = fetchUser() || {};
```

### Mistake 3 — Object.keys() missing inherited or Symbol properties

```javascript
// Object.keys() returns ONLY own + enumerable + string-keyed properties
const parent = { inherited: true };
const child = Object.create(parent);
child.own = true;
Object.keys(child); // ['own'] — 'inherited' not included

// For ALL own (including non-enumerable):
Object.getOwnPropertyNames(child); // ['own']
Object.getOwnPropertySymbols(child); // symbol-keyed properties
Reflect.ownKeys(child); // all own properties (strings + symbols)
```

---

## 12. Exercises

### Exercise 1 — Deep merge

```javascript
// Write a function that deep-merges two objects.
// deepMerge({ a: 1, b: { c: 2 } }, { b: { d: 3 }, e: 4 })
// → { a: 1, b: { c: 2, d: 3 }, e: 4 }
```

<details>
<summary>Solution</summary>

```javascript
function deepMerge(target, source) {
  const result = { ...target };
  for (const key of Object.keys(source)) {
    if (
      typeof source[key] === "object" &&
      source[key] !== null &&
      !Array.isArray(source[key]) &&
      typeof target[key] === "object" &&
      target[key] !== null
    ) {
      result[key] = deepMerge(target[key], source[key]);
    } else {
      result[key] = source[key];
    }
  }
  return result;
}
```

</details>

---

## 🔗 Related Topics

- [`06-prototypes.md`](./06-prototypes.md) — How object inheritance works via prototype chain
- [`24-es6-modern-syntax.md`](./24-es6-modern-syntax.md) — More modern object features
- [`27-proxy-reflect-and-metaprogramming.md`](./27-proxy-reflect-and-metaprogramming.md) — Intercepting object operations
