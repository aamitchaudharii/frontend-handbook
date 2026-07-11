# 06 — Type Narrowing

> **"TypeScript's type narrowing is the mechanism by which a broad union type becomes a specific, usable type inside a conditional block. Understanding narrowing means understanding how TypeScript reads your runtime checks and uses them to refine types — the same way a human would reason about code."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [What Is Narrowing?](#1-what-is-narrowing)
2. [typeof Guards](#2-typeof-guards)
3. [Truthiness Narrowing](#3-truthiness-narrowing)
4. [Equality Narrowing](#4-equality-narrowing)
5. [in Operator Narrowing](#5-in-operator-narrowing)
6. [instanceof Narrowing](#6-instanceof-narrowing)
7. [Type Predicates (is)](#7-type-predicates-is)
8. [Assertion Functions (asserts)](#8-assertion-functions-asserts)
9. [Discriminated Union Narrowing](#9-discriminated-union-narrowing)
10. [Control Flow Analysis](#10-control-flow-analysis)
11. [Common Mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)

---

## 1. What Is Narrowing?

```typescript
// TypeScript tracks types through control flow.
// When you check a value, TypeScript narrows its type inside that branch.

function example(value: string | number) {
  //  here: value is string | number

  if (typeof value === "string") {
    //  here: value is string (narrowed)
    value.toUpperCase(); // ✅
  } else {
    //  here: value is number (narrowed — only remaining option)
    value.toFixed(2); // ✅
  }

  //  here: value is string | number again (back to full union)
}

// TypeScript narrows through:
// - typeof checks
// - equality checks (=== / !==)
// - truthiness checks (if (value) ...)
// - `in` operator
// - instanceof
// - type predicates (user-defined)
// - assignment
// - control flow (early return, throw)
```

---

## 2. typeof Guards

```typescript
// typeof returns a string — TypeScript narrows based on string comparison
type Primitive = string | number | boolean | bigint | symbol | null | undefined;

function process(x: string | number | boolean | null | undefined) {
  if (typeof x === "string") {
    x.toUpperCase(); // x: string
  } else if (typeof x === "number") {
    x.toFixed(2); // x: number
  } else if (typeof x === "boolean") {
    x.valueOf(); // x: boolean
  } else {
    x; // x: null | undefined
  }
}

// typeof limitations:
// typeof null === 'object' ← the historical bug — check === null separately
// typeof [] === 'object'   ← arrays are objects
// typeof function(){} === 'function' ← functions are detectable

function handleValue(x: string | number | null | object) {
  if (x === null) {
    // x: null (equality check first)
  } else if (typeof x === "object") {
    // x: object (now we know it's NOT null)
    Object.keys(x);
  } else if (typeof x === "string") {
    x.length;
  } else {
    x; // x: number
  }
}
```

---

## 3. Truthiness Narrowing

```typescript
// Falsy values: false, 0, -0, 0n, '', null, undefined, NaN
// TypeScript narrows based on truthiness

function greet(name: string | null | undefined) {
  if (name) {
    // name is string (null and undefined are falsy, so filtered out)
    // BUT: empty string '' is also falsy — excluded too!
    name.toUpperCase(); // ✅
  } else {
    // name is string | null | undefined (could be '')
  }
}

// ⚠️ Truthiness narrows out falsy values INCLUDING ''  and 0
function processCount(count: number | null) {
  if (count) {
    count; // number — but 0 is excluded! even though 0 is a valid count
  }
}

// ✅ Use explicit null check instead:
function processCount2(count: number | null) {
  if (count !== null) {
    count; // number — includes 0 ✅
  }
}

// Double negation (!!) narrows to boolean
function hasValue<T>(x: T): x is NonNullable<T> {
  return x != null;
}
```

---

## 4. Equality Narrowing

```typescript
// === and !== narrow precisely
function example(x: string | number, y: string | boolean) {
  if (x === y) {
    // x and y must both be string (only overlap between string|number and string|boolean)
    x.toUpperCase(); // x: string
    y.toUpperCase(); // y: string
  }
}

// Literal equality narrows to that literal
type Status = "active" | "inactive" | "pending";
function handleStatus(s: Status) {
  if (s === "active") {
    s; // s: 'active'
  } else {
    s; // s: 'inactive' | 'pending'
  }
}

// switch narrows per case
function describe(x: string | number | boolean): string {
  switch (typeof x) {
    case "string":
      return x.toUpperCase(); // x: string
    case "number":
      return x.toFixed(2); // x: number
    case "boolean":
      return x ? "yes" : "no"; // x: boolean
  }
}

// null/undefined checks
function use(x: string | null | undefined) {
  if (x !== null && x !== undefined) {
    x.length;
  } // x: string
  if (x != null) {
    x.length;
  } // same — != null catches both
  x?.length; // optional chaining
}
```

---

## 5. in Operator Narrowing

```typescript
// `prop in obj` narrows to types that have that property

type Cat = { kind: "cat"; meow(): void };
type Dog = { kind: "dog"; bark(): void };
type Pet = Cat | Dog;

function makeSound(pet: Pet) {
  if ("meow" in pet) {
    pet.meow(); // pet: Cat ✅
  } else {
    pet.bark(); // pet: Dog ✅
  }
}

// `in` with optional properties
type A = { x: string; y?: number };
type B = { x: string; z: boolean };
type AB = A | B;

function handle(val: AB) {
  if ("z" in val) {
    val; // A | B (could still be A with z added externally)
    val.z; // boolean ✅ (val has z, must satisfy B)
  }
  if ("y" in val) {
    val.y; // number | undefined (y is optional in A)
  }
}

// `in` for unknown values (runtime type checking)
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" && obj !== null && "id" in obj && "name" in obj
  );
}
```

---

## 6. instanceof Narrowing

```typescript
// instanceof narrows to the class type
class Circle {
  constructor(public radius: number) {}
}
class Rectangle {
  constructor(
    public w: number,
    public h: number,
  ) {}
}

function area(shape: Circle | Rectangle): number {
  if (shape instanceof Circle) {
    return Math.PI * shape.radius ** 2; // shape: Circle
  }
  return shape.w * shape.h; // shape: Rectangle
}

// Works with built-in classes too
function formatError(err: unknown): string {
  if (err instanceof Error) {
    return err.message; // err: Error
  }
  if (err instanceof TypeError) {
    return `Type error: ${err.message}`; // err: TypeError
  }
  return String(err);
}

// instanceof and prototype chain
class Animal {}
class Dog extends Animal {}
const d = new Dog();
d instanceof Dog; // true
d instanceof Animal; // true (prototype chain)
```

---

## 7. Type Predicates (is)

```typescript
// A type predicate is a function that returns `value is Type`
// It tells TypeScript: "if this returns true, the argument is this type"

function isString(x: unknown): x is string {
  return typeof x === "string";
}

function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" &&
    obj !== null &&
    typeof (obj as any).id === "number" &&
    typeof (obj as any).name === "string"
  );
}

// Usage: TypeScript narrows based on the predicate's return
const data: unknown = fetchData();
if (isUser(data)) {
  data.name; // ✅ data is User inside this block
}

// Filter with type predicate (removes null/undefined from arrays)
const maybes: (string | null)[] = ["Alice", null, "Bob", null];
const names: string[] = maybes.filter((x): x is string => x !== null);
//                                         ^^^^^^^^^^^ type predicate in filter

// NonNullable predicate
function isDefined<T>(x: T | null | undefined): x is NonNullable<T> {
  return x !== null && x !== undefined;
}
const results = [1, null, 2, undefined, 3].filter(isDefined); // number[]
```

---

## 8. Assertion Functions (asserts)

```typescript
// An assertion function throws if a condition is NOT met
// TypeScript narrows based on the asserted condition after the call

function assert(condition: boolean, msg?: string): asserts condition {
  if (!condition) throw new Error(msg ?? "Assertion failed");
}

function assertIsString(x: unknown): asserts x is string {
  if (typeof x !== "string")
    throw new TypeError(`Expected string, got ${typeof x}`);
}

// Usage:
function processInput(input: unknown) {
  assertIsString(input);
  // After this point, TypeScript knows input is string
  input.toUpperCase(); // ✅ no narrowing block needed
}

function loadConfig(data: unknown) {
  assert(typeof data === "object" && data !== null, "Expected object");
  // data is now object (not null)
  assert("host" in data, "Missing host");
  // data has 'host' property
}

// Combining: assert + type predicate
function assertDefined<T>(
  x: T | null | undefined,
  name: string,
): asserts x is T {
  if (x == null) throw new Error(`${name} must not be null/undefined`);
}
```

---

## 9. Discriminated Union Narrowing

```typescript
// TypeScript automatically narrows discriminated unions in switch/if
type Result<T> =
  | { status: "success"; data: T }
  | { status: "error"; error: Error }
  | { status: "loading" };

function render<T>(result: Result<T>): string {
  // switch on discriminant: TypeScript narrows each case
  switch (result.status) {
    case "success":
      return JSON.stringify(result.data); // result.data: T ✅
    case "error":
      return result.error.message; // result.error: Error ✅
    case "loading":
      return "Loading...";
  }
}

// if/else also works
function handle<T>(result: Result<T>) {
  if (result.status === "success") {
    result.data; // T
  } else if (result.status === "error") {
    result.error; // Error
  } else {
    result; // { status: 'loading' }
  }
}
```

---

## 10. Control Flow Analysis

```typescript
// TypeScript tracks types through the ENTIRE control flow, not just if-branches

function process(x: string | null): string {
  if (x === null) {
    return "default"; // ← early return
  }
  // TypeScript knows x is NOT null here (it would have returned above)
  return x.toUpperCase(); // x: string ✅
}

// After assignment, type is narrowed to the assigned type
let value: string | number;
value = "hello";
value.toUpperCase(); // value: string ✅ (narrowed by assignment)
value = 42;
value.toFixed(); // value: number ✅

// Narrowing persists through closures... with caveats
function tricky(x: string | null) {
  if (x !== null) {
    // x: string here
    setTimeout(() => {
      x; // x: string | null ← TypeScript conservatively widens in callbacks
      // because x could be reassigned before the callback runs
    }, 0);
  }
}

// Use a const to preserve the narrowed type in closures
function fixed(x: string | null) {
  if (x !== null) {
    const str = x; // str: string (const — can't be reassigned)
    setTimeout(() => {
      str; // str: string ✅ — const can't be widened
    }, 0);
  }
}

// Unreachable code detection
function alwaysThrows(): never {
  throw new Error();
}

function example(x: string | number) {
  if (typeof x === "string") {
    alwaysThrows();
    x.toUpperCase(); // ← TS marks as unreachable code (after never-returning fn)
  }
}
```

---

## 11. Common Mistakes

### Mistake 1 — Narrowing doesn't persist across function calls

```typescript
// ❌ TS doesn't know what happens inside the function call
function isNonNull<T>(x: T | null): boolean {
  return x !== null;
}

let x: string | null = "hello";
if (isNonNull(x)) {
  x.toUpperCase(); // ❌ still string | null — TS doesn't look inside isNonNull
}

// ✅ Use a type predicate instead
function isNonNull<T>(x: T | null): x is T {
  return x !== null;
}
if (isNonNull(x)) {
  x.toUpperCase(); // ✅ x is string
}
```

### Mistake 2 — Reassignment widens the type

```typescript
let x: string | null = getStringOrNull();
if (x !== null) {
  x; // x: string
  x = getStringOrNull(); // assignment widens back to string | null
  x; // x: string | null again
}
```

---

## 12. Exercises

### Exercise 1 — Write a type-safe `parseJSON`

```typescript
// Write parseJSON that:
// - Takes a string
// - Returns [null, T] on success (where T is a generic)
// - Returns [Error, null] on failure
// - The caller uses a type predicate to check if result is T
```

<details>
<summary>Solution</summary>

```typescript
function parseJSON<T>(json: string): [null, T] | [Error, null] {
  try {
    return [null, JSON.parse(json) as T];
  } catch (e) {
    return [e instanceof Error ? e : new Error(String(e)), null];
  }
}

const [err, data] = parseJSON<{ name: string }>('{"name":"Alice"}');
if (err) {
  console.error(err.message);
} else {
  data.name; // ✅ data is { name: string }
}
```

</details>

---

## 🔗 Related Topics

- [`05-union-and-intersection-types.md`](./05-union-and-intersection-types.md) — Discriminated unions
- [`07-generics.md`](./07-generics.md) — Generic type predicates
