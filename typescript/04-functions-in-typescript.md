# 04 — Functions in TypeScript

> **"In TypeScript, a function's type is its contract: what goes in, what comes out, and what's optional. Getting function types right pays off everywhere — callbacks, higher-order functions, overloads — because a precisely typed function makes every caller correct by construction."**

🟢 **Level: Beginner**

---

## 📚 Table of Contents

1. [Parameter and Return Types](#1-parameter-and-return-types)
2. [Optional and Default Parameters](#2-optional-and-default-parameters)
3. [Rest Parameters](#3-rest-parameters)
4. [Function Types and Signatures](#4-function-types-and-signatures)
5. [void vs never vs undefined](#5-void-vs-never-vs-undefined)
6. [Function Overloads](#6-function-overloads)
7. [this in Functions](#7-this-in-functions)
8. [Generic Functions (Introduction)](#8-generic-functions-introduction)
9. [Common Mistakes](#9-common-mistakes)
10. [Exercises](#10-exercises)

---

## 1. Parameter and Return Types

```typescript
// Basic annotation
function add(a: number, b: number): number {
  return a + b;
}

// Arrow function
const multiply = (a: number, b: number): number => a * b;

// Return type is usually inferred — annotating it is optional but recommended
// for public API functions (documents intent, catches errors)
function getUser(id: number) {
  // inferred return: User | undefined
  return users.find((u) => u.id === id);
}

function getUser(id: number): User {
  // explicit — TypeScript will error if
  return users.find((u) => u.id === id)!; // return value doesn't match
}

// Object parameter type
function createUser(data: { name: string; email: string }): User {
  return { id: nextId++, ...data };
}

// Destructured parameter type
function display({ name, age = 0 }: { name: string; age?: number }): string {
  return `${name} (${age})`;
}
```

---

## 2. Optional and Default Parameters

```typescript
// Optional parameter: may be omitted, type becomes T | undefined
function greet(name: string, title?: string): string {
  return title ? `${title} ${name}` : name;
}
greet("Alice"); // OK
greet("Alice", "Dr."); // OK
// greet();             // ❌ name is required

// Default parameter: provides fallback, parameter is optional
function connect(host: string, port: number = 5432): string {
  return `${host}:${port}`;
}
connect("localhost"); // 'localhost:5432'
connect("localhost", 3306); // 'localhost:3306'
connect("localhost", undefined); // 'localhost:5432' (undefined triggers default)

// Optional vs default:
// Optional (?) → type is T | undefined inside the function
// Default      → type is T inside the function (no need to handle undefined)
function optEx(x?: number) {
  x;
} // x is number | undefined
function defEx(x: number = 0) {
  x;
} // x is number

// Optional parameters MUST come after required ones
// function bad(a?: string, b: string) {} // ❌ required after optional
function good(a: string, b?: string) {} // ✅
```

---

## 3. Rest Parameters

```typescript
// Collects remaining arguments as a typed array
function sum(...numbers: number[]): number {
  return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4, 5); // 15

// With leading required parameters
function log(level: "info" | "warn" | "error", ...messages: string[]): void {
  console.log(`[${level}]`, ...messages);
}
log("info", "Server started", "on port 3000");

// Typed rest as a tuple (precise argument types)
function tagged(first: string, ...rest: [number, boolean]): void {
  const [count, active] = rest;
}
tagged("hello", 42, true); // ✅
// tagged('hello', 'world', true); // ❌ second arg must be number

// Spread as arguments
const args: [string, number] = ["Alice", 30];
function createUser(name: string, age: number) {
  return { name, age };
}
createUser(...args); // ✅ tuple spread is type-safe
```

---

## 4. Function Types and Signatures

```typescript
// Type alias for a function
type Predicate<T> = (item: T) => boolean;
type Transformer<T, U> = (input: T) => U;
type EventHandler = (event: Event) => void;
type AsyncLoader<T> = (id: string) => Promise<T>;

// Using function type aliases
const isEven: Predicate<number> = (n) => n % 2 === 0;
const double: Transformer<number, number> = (n) => n * 2;

// Call signature in an interface
interface Comparator<T> {
  (a: T, b: T): number;
}
const numSort: Comparator<number> = (a, b) => a - b;
[3, 1, 2].sort(numSort);

// Function with properties (callable object)
interface LogFunction {
  (message: string): void;
  level: "debug" | "info" | "warn" | "error";
  setLevel(level: this["level"]): void;
}

// Higher-order function types
function compose<A, B, C>(f: (b: B) => C, g: (a: A) => B): (a: A) => C {
  return (a) => f(g(a));
}
const doubleString = compose(
  (n: number) => String(n),
  (s: string) => s.length,
);
doubleString("hello"); // "5"

// Callback types
function fetchAndProcess(
  url: string,
  onSuccess: (data: unknown) => void,
  onError: (error: Error) => void,
): void {
  fetch(url)
    .then((r) => r.json())
    .then(onSuccess)
    .catch(onError);
}
```

---

## 5. void vs never vs undefined

```typescript
// void: function returns, but the return value isn't meaningful
function log(msg: string): void {
  console.log(msg);
  // returns undefined implicitly
}
const result = log("hi"); // result is void — you shouldn't use it

// A void-returning function CAN return undefined explicitly
function voidEx(): void {
  return; // ✅ OK
  return undefined; // ✅ OK
  // return 42;    // ❌ Error
}

// never: function NEVER returns (throws or infinite loops)
function fail(msg: string): never {
  throw new Error(msg);
}
function loop(): never {
  while (true) {}
}

// never in union types: it's the identity element (filtered out)
type A = string | never; // just string
type B = string & never; // never (impossible to satisfy both)

// undefined as return type: function returns undefined explicitly
function explicit(): undefined {
  return undefined; // must return undefined explicitly
}

// Practical distinction:
// void   → callback that returns something that will be IGNORED
// never  → function that truly never returns
// undefined → function that explicitly returns undefined as meaningful
const arr = [1, 2, 3];
arr.forEach((n) => {
  // forEach expects (n: number) => void
  return n * 2; // ✅ fine — return value from void fn is ignored
});
```

---

## 6. Function Overloads

```typescript
// Overloads: same function name, different type signatures
// Provide MULTIPLE call signatures, then one implementation

// Overload signatures (no body):
function format(value: string): string;
function format(value: number): string;
function format(value: Date): string;
// Implementation signature (must be compatible with ALL overloads):
function format(value: string | number | Date): string {
  if (typeof value === "string") return value.toUpperCase();
  if (typeof value === "number") return value.toFixed(2);
  return value.toISOString();
}

format("hello"); // TypeScript knows: returns string
format(3.14); // TypeScript knows: returns string
format(new Date()); // TypeScript knows: returns string

// Return type overloads (different return types per input):
function createElement(tag: "canvas"): HTMLCanvasElement;
function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: "span"): HTMLSpanElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}
const canvas = createElement("canvas"); // typed as HTMLCanvasElement ✅
const div = createElement("div"); // typed as HTMLDivElement ✅

// When to use overloads:
// ✅ Different input types → different output types
// ✅ Conditional argument requirements (some args only valid with others)
// ❌ When a simple union or generic works just as well
```

---

## 7. this in Functions

```typescript
// TypeScript can type `this` via a special first parameter (erased at runtime)
interface DB {
  connection: string;
  query(this: DB, sql: string): unknown[];
}

const db: DB = {
  connection: "postgres://...",
  query(this: DB, sql: string) {
    console.log(this.connection); // TS knows `this` is DB
    return [];
  },
};

// NoInfer<this> pattern for fluent builders
class QueryBuilder {
  private clauses: string[] = [];

  where(this: this, clause: string): this {
    // `this` return type = subclass type
    this.clauses.push(clause);
    return this;
  }
  build(): string {
    return this.clauses.join(" AND ");
  }
}

// thisParameterType and OmitThisParameter utility types
type QueryFn = (this: DB, sql: string) => unknown[];
type WithoutThis = OmitThisParameter<QueryFn>; // (sql: string) => unknown[]
```

---

## 8. Generic Functions (Introduction)

```typescript
// A function that works on ANY type while preserving type information
function identity<T>(value: T): T {
  return value;
}
identity("hello"); // T inferred as string, returns string
identity(42); // T inferred as number, returns number
identity({ x: 1 }); // T inferred as { x: number }

// Generic with constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const user = { name: "Alice", age: 30 };
getProperty(user, "name"); // returns string
getProperty(user, "age"); // returns number
// getProperty(user, 'email'); // ❌ 'email' is not a key of user

// Generic array utility
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
first([1, 2, 3]); // number | undefined
first(["a", "b"]); // string | undefined
first([]); // undefined

// See 07-generics.md for full coverage
```

---

## 9. Common Mistakes

### Mistake 1 — Implicit any on parameters

```typescript
// ❌ With noImplicitAny: true (which strict enables), this is an error
function process(data) {
  // Parameter 'data' implicitly has an 'any' type
  return data.name;
}

// ✅ Always type parameters
function process(data: { name: string }) {
  return data.name;
}
```

### Mistake 2 — Callback return type mismatch

```typescript
// ❌ Returning a value from a void callback isn't an error but can be confusing
const nums = [1, 2, 3];
const result = nums.forEach((n) => n * 2); // result is void — the values are lost!

// ✅ Use map when you need the transformed values
const doubled = nums.map((n) => n * 2); // [2, 4, 6]
```

### Mistake 3 — Overload implementation visible to callers

```typescript
function parse(x: string): number;
function parse(x: number): string;
function parse(x: string | number): number | string {
  // implementation
  return typeof x === "string" ? Number(x) : String(x);
}

// The IMPLEMENTATION signature is NOT visible to callers
// parse('42' as string | number); // ❌ Error — 'string | number' isn't an overload
// Only the overload signatures are callable
parse("42"); // ✅
parse(42); // ✅
```

---

## 10. Exercises

### Exercise 1 — Type a retry function

```typescript
// Type this retry utility so that:
// 1. fn can return any type T
// 2. The return type of retry is Promise<T>
// 3. maxAttempts has a default of 3
async function retry(fn, maxAttempts = 3) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === maxAttempts - 1) throw e;
    }
  }
}
```

<details>
<summary>Solution</summary>

```typescript
async function retry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
): Promise<T> {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === maxAttempts - 1) throw e;
    }
  }
  throw new Error("retry: unreachable"); // satisfies return type
}

// Usage:
const data = await retry(() => fetch("/api/data").then((r) => r.json()));
// data is typed as unknown (whatever fetch returns)
```

</details>

---

## 🔗 Related Topics

- [`07-generics.md`](./07-generics.md) — Full generics coverage
- [`08-utility-types.md`](./08-utility-types.md) — ReturnType, Parameters utility types
- [`javascript-core/19-functions-fundamentals.md`](../javascript-core/19-functions-fundamentals.md) — JS function fundamentals
