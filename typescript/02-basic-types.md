# 02 — Basic Types

> **"The TypeScript type system has one job: describe the shape of data. Once you can describe any shape — a primitive, an array, a nested object, a function — you can reason about code the way a compiler does. Everything after this is refinement of that one skill."**

🟢 **Level: Beginner**

---

## 📚 Table of Contents

1. [Primitive Types](#1-primitive-types)
2. [Arrays and Tuples](#2-arrays-and-tuples)
3. [Object Types](#3-object-types)
4. [Special Types — any, unknown, never, void](#4-special-types--any-unknown-never-void)
5. [Type Aliases](#5-type-aliases)
6. [Literal Types](#6-literal-types)
7. [Union Types (introduction)](#7-union-types-introduction)
8. [Type Assertions](#8-type-assertions)
9. [Common Mistakes](#9-common-mistakes)
10. [Exercises](#10-exercises)

---

## 1. Primitive Types

```typescript
// The 7 JS primitives — TypeScript knows all of them
let name: string = "Alice";
let age: number = 30; // integers AND floats — both are 'number'
let active: boolean = true;
let nothing: null = null;
let missing: undefined = undefined;
let id: symbol = Symbol("id");
let bigNum: bigint = 9007199254740993n;

// TypeScript adds:
// never, void, any, unknown (covered in Section 4)

// IMPORTANT: null and undefined are NOT in other types with strictNullChecks: true
let n: number = null; // ❌ Error with strictNullChecks
let n: number | null = null; // ✅ explicit

// object type: anything that isn't a primitive
let obj: object = { name: "Alice" };
// But `object` is rarely useful — prefer specific object shapes
```

---

## 2. Arrays and Tuples

```typescript
// Arrays: all elements same type
const names: string[] = ["Alice", "Bob"];
const scores: number[] = [95, 87];
const flags: boolean[] = [true, false];
const mixed: (string | number)[] = ["Alice", 30]; // union element type

// Generic array syntax (equivalent)
const names2: Array<string> = ["Alice", "Bob"];

// Readonly arrays (can't push, pop, etc.)
const fixed: readonly number[] = [1, 2, 3];
const fixed2: ReadonlyArray<number> = [1, 2, 3]; // equivalent
fixed.push(4); // ❌ Error: push does not exist on readonly array

// Tuples: fixed-length arrays where each position has a known type
let point: [number, number] = [0, 0];
let entry: [string, number] = ["age", 30];

point[0]; // number
point[2]; // ❌ Error: tuple index 2 out of bounds

// Named tuple elements (TS 4.0+)
type Range = [start: number, end: number];
type RGB = [red: number, green: number, blue: number];

// Rest elements in tuples
type StringsThenNumber = [...string[], number];
const ex: StringsThenNumber = ["a", "b", "c", 42]; // last element is number

// Optional tuple elements
type Point3D = [number, number, number?]; // z is optional
const p2d: Point3D = [0, 0];
const p3d: Point3D = [0, 0, 5];
```

---

## 3. Object Types

```typescript
// Inline object type
function greet(user: { name: string; age: number }): string {
  return `${user.name}, ${user.age}`;
}

// Optional properties
type Config = {
  host: string;
  port?: number; // optional: number | undefined
  debug?: boolean;
};

// Readonly properties
type Point = {
  readonly x: number;
  readonly y: number;
};
const p: Point = { x: 0, y: 0 };
p.x = 5; // ❌ Error: Cannot assign to 'x' because it is a read-only property

// Index signatures: when keys are dynamic but values are a known type
type StringMap = { [key: string]: string };
const headers: StringMap = { "Content-Type": "application/json" };

type NumberRecord = { [key: string]: number };
const scores: NumberRecord = { alice: 95, bob: 87 };

// Combining known + dynamic properties
type Config2 = {
  name: string; // always present
  [key: string]: string; // additional arbitrary string properties
  // Note: all values including 'name' must satisfy the index signature type (string)
};

// Nested objects
type User = {
  name: string;
  address: {
    street: string;
    city: string;
    zip?: string;
  };
};
```

---

## 4. Special Types — any, unknown, never, void

```typescript
// ═══ any ═══
// Completely disables type checking for this value
// The escape hatch — avoid when possible
let x: any = "hello";
x = 42; // OK
x.anything(); // OK — no type error even if it crashes at runtime
x as number; // no error

// ═══ unknown ═══
// Like any, but SAFE — you can't use it without narrowing first
let val: unknown = fetchData();

val.toUpperCase(); // ❌ Error — can't call methods on unknown
val + 1; // ❌ Error

// Must narrow before use:
if (typeof val === "string") {
  val.toUpperCase(); // ✅ OK inside this block
}
// ✅ Prefer unknown over any when type is genuinely unknown (e.g., API responses)

// ═══ never ═══
// A value that NEVER EXISTS — the bottom type
// No value can be assigned to never (except never itself)
function throwError(msg: string): never {
  throw new Error(msg); // function never returns
}
function infinite(): never {
  while (true) {} // never returns
}

// PRACTICAL USE: exhaustiveness checking
type Shape = "circle" | "square" | "triangle";
function area(shape: Shape): number {
  switch (shape) {
    case "circle":
      return Math.PI;
    case "square":
      return 1;
    case "triangle":
      return 0.5;
    default:
      const _exhaustive: never = shape; // ❌ Error if a new Shape was added but not handled
      throw new Error(`Unknown shape: ${shape}`);
  }
}

// ═══ void ═══
// The return type of functions that don't return a meaningful value
function log(msg: string): void {
  console.log(msg);
  // implicit return undefined
}

// void vs never:
// void: the function returns (with undefined)
// never: the function NEVER returns (throws or infinite loops)

// ═══ undefined and null ═══
let u: undefined = undefined;
let n: null = null;
// With strictNullChecks: these are only assignable to themselves (and each other)
// Without strictNullChecks: assignable to anything (dangerous — always use strict)
```

---

## 5. Type Aliases

```typescript
// Give a name to any type expression
type ID = string | number;
type Name = string;
type Point = { x: number; y: number };
type Matrix = number[][];

// Use the alias anywhere you'd use the type:
function getUser(id: ID): void {
  /* ... */
}
const origin: Point = { x: 0, y: 0 };

// Type aliases for function types
type Predicate<T> = (item: T) => boolean;
type Transform<T, U> = (input: T) => U;

const isEven: Predicate<number> = (n) => n % 2 === 0;
const toString: Transform<number, string> = (n) => String(n);

// Type aliases are NOT classes — no runtime existence
// They are ERASED at compile time

// Recursive type aliases
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };
```

---

## 6. Literal Types

```typescript
// A type that is exactly ONE specific value
let a: "hello" = "hello";
a = "world"; // ❌ Error: 'world' is not assignable to type 'hello'

let b: 42 = 42;
let c: true = true;

// PRACTICAL: literal union types (much better than plain string)
type Direction = "north" | "south" | "east" | "west";
type Status = "pending" | "active" | "inactive" | "deleted";
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

function move(direction: Direction): void {
  /* ... */
}
move("north"); // ✅
move("up"); // ❌ Error: 'up' is not assignable to type 'Direction'

// as const: lock an object's values to their literal types
const config = {
  host: "localhost",
  port: 3000,
  debug: false,
} as const;
// config.host is 'localhost' (not string)
// config.port is 3000 (not number)
// All properties are readonly

// as const on arrays: makes a tuple of literal types
const methods = ["GET", "POST", "PUT"] as const;
// Type: readonly ["GET", "POST", "PUT"]
type HttpMethods = (typeof methods)[number]; // "GET" | "POST" | "PUT"
```

---

## 7. Union Types (introduction)

```typescript
// A value that can be ONE OF several types
type StringOrNumber = string | number;

function format(value: string | number): string {
  if (typeof value === "number") {
    return value.toFixed(2);
  }
  return value.toUpperCase();
}

// Union with null/undefined (common pattern with strictNullChecks)
type MaybeString = string | null;
type OptionalUser = User | undefined;

function greet(name: string | null): string {
  return `Hello, ${name ?? "Guest"}`;
}

// Arrays of unions
const mixed: (string | number)[] = ["Alice", 42, "Bob", 30];

// See 05-union-and-intersection-types.md for discriminated unions and more
```

---

## 8. Type Assertions

```typescript
// Tell TypeScript "trust me, I know the type"
const input = document.getElementById("email") as HTMLInputElement;
input.value; // TS now knows HTMLInputElement has .value

// Double assertion (use sparingly — bypasses safety)
const x = value as unknown as number; // force any type

// Non-null assertion (!)
const el = document.getElementById("root")!; // asserts non-null/undefined
// ⚠️ If el IS null, this crashes at runtime with no warning from TS

// satisfies operator (TS 4.9+): check type without widening it
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies Record<string, string | number[]>;
// palette.red is still number[] (not string | number[])
// satisfies validates the shape WITHOUT losing the precise inferred type

// When to use assertions:
// ✅ DOM access (you know the element type better than TS does)
// ✅ JSON parsing results (TS types them as any)
// ✅ When TS inference is too wide and you can't fix it at the source
// ❌ To silence errors you don't understand — fix the types instead
```

---

## 9. Common Mistakes

### Mistake 1 — Object type vs `object` vs `{}`

```typescript
let a: object = { name: "Alice" }; // any non-primitive — barely useful
let b: {} = 42; // literally anything non-null — almost useless
let c: { name: string } = { name: "Alice" }; // ✅ specific shape

// ✅ Always prefer specific object shapes over `object` or `{}`
```

### Mistake 2 — Forgetting that tuples are not enforced at runtime

```typescript
type Point = [number, number];
const p: Point = [0, 0];

// At runtime, p is just a regular JavaScript array
// You could push a third element at runtime and TS wouldn't know
(p as any).push(99); // bypasses TS — p is now [0, 0, 99] at runtime

// TypeScript's tuple type is a compile-time check, not a runtime constraint
```

### Mistake 3 — `never` in unexpected positions

```typescript
// If TS infers a variable as `never`, it usually means your code has a logic conflict
type A = string & number; // never — no value can be both
function f(): never {
  // if you reach code after a throw/return, TS may infer unreachable code as never
}
// When you see `never` unexpectedly, look for an impossible intersection or condition
```

---

## 10. Exercises

### Exercise 1 — Type a configuration object

```typescript
// Add appropriate TypeScript types to this configuration object and the
// function that uses it. Use readonly where values shouldn't change after init.
const serverConfig = {
  host: "localhost",
  port: 3000,
  allowedOrigins: ["http://localhost:5173"],
  ssl: false,
};

function startServer(config) {
  // config.host should be readable but not modifiable
  // config.port should be a number
  // config.allowedOrigins is an array of strings
}
```

<details>
<summary>Solution</summary>

```typescript
type ServerConfig = {
  readonly host: string;
  readonly port: number;
  readonly allowedOrigins: readonly string[];
  readonly ssl: boolean;
};

const serverConfig: ServerConfig = {
  host: "localhost",
  port: 3000,
  allowedOrigins: ["http://localhost:5173"],
  ssl: false,
};

function startServer(config: ServerConfig): void {
  console.log(`Starting on ${config.host}:${config.port}`);
  // config.port = 8080; // ❌ Error: Cannot assign to 'port' (readonly)
}
```

</details>

---

## 🔗 Related Topics

- [`03-interfaces-and-type-aliases.md`](./03-interfaces-and-type-aliases.md) — Structuring complex types
- [`05-union-and-intersection-types.md`](./05-union-and-intersection-types.md) — Combining types
- [`javascript-core/16-variables-and-data-types.md`](../javascript-core/16-variables-and-data-types.md) — JS types this builds on
