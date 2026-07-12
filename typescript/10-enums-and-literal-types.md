# 10 — Enums and Literal Types

> **"Enums and literal union types solve the same problem — representing a fixed set of meaningful values — but they solve it differently. Enums exist at runtime and carry historical baggage. Literal unions are pure types, erased at compile time, with better inference and fewer surprises. Knowing when to use each, and when to prefer `as const`, separates TypeScript novices from experienced users."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [Numeric Enums](#1-numeric-enums)
2. [String Enums](#2-string-enums)
3. [const Enums](#3-const-enums)
4. [Enum Pitfalls](#4-enum-pitfalls)
5. [Literal Types](#5-literal-types)
6. [as const — The Modern Alternative](#6-as-const--the-modern-alternative)
7. [Discriminated Unions with Literals](#7-discriminated-unions-with-literals)
8. [Template Literal Types](#8-template-literal-types)
9. [Enum vs Literal Union — When to Use Each](#9-enum-vs-literal-union--when-to-use-each)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. Numeric Enums

```typescript
// Numeric enum: members get auto-incremented numbers by default
enum Direction {
  North, // 0
  South, // 1
  East, // 2
  West, // 3
}

Direction.North; // 0
Direction[0]; // 'North' — reverse mapping! (numeric enums only)

// Custom values
enum StatusCode {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  Forbidden = 403,
  NotFound = 404,
  InternalError = 500,
}

StatusCode.OK; // 200
StatusCode[200]; // 'OK' (reverse mapping)

// Members can reference earlier members
enum FileAccess {
  None,
  Read = 1 << 0, // 1
  Write = 1 << 1, // 2
  ReadWrite = Read | Write, // 3
}

// Bit flags pattern (combining enum values)
function canRead(access: FileAccess): boolean {
  return (access & FileAccess.Read) !== 0;
}
canRead(FileAccess.ReadWrite); // true
```

---

## 2. String Enums

```typescript
// String enum: no reverse mapping, more readable in logs/debugging
enum Status {
  Active = "active",
  Inactive = "inactive",
  Pending = "pending",
  Deleted = "deleted",
}

Status.Active; // 'active' (the string value)
// Status['active']; // ❌ no reverse mapping for string enums

// Heterogeneous enum (mixed types — generally avoid)
enum Mixed {
  Yes = 1,
  No = "NO", // legal but confusing
}

// String enums in practice
function setStatus(status: Status): void {
  fetch(`/api/status`, { method: "PUT", body: JSON.stringify({ status }) });
  // status is 'active', 'inactive', etc. in the request body
}
setStatus(Status.Active); // ✅ sends 'active'
// setStatus('active');     // ❌ string not assignable to Status (type-safe!)
```

---

## 3. const Enums

```typescript
// const enum: inlined at compile time — NO runtime object generated
const enum Direction {
  North,
  South,
  East,
  West,
}

// TypeScript replaces uses with the literal value:
const d = Direction.North; // compiled to: const d = 0; (no Direction object in JS output)

// ✅ Zero runtime cost
// ❌ Cannot use as a value at runtime (Direction.North exists only in TS, not JS)
// ❌ Cannot iterate over const enum members at runtime
// ❌ Problematic when used across package boundaries (declaration files)

// When to use const enum:
// - Same package, numeric values, no runtime reflection needed
// - Example: performance-critical flags, shader constants

const enum Flags {
  None = 0,
  Read = 1 << 0,
  Write = 1 << 1,
}

function check(flags: number, flag: Flags): boolean {
  return (flags & flag) !== 0;
}
// Compiles to: return (flags & 1) !== 0; (Flags.Read is inlined as 1)
```

---

## 4. Enum Pitfalls

```typescript
// PITFALL 1: Numeric enums accept any number (structural typing issue)
enum Color {
  Red = 0,
  Green = 1,
  Blue = 2,
}
function paintWall(color: Color): void {
  /* ... */
}
paintWall(42); // ✅ TypeScript doesn't error — any number satisfies Color!
paintWall(Color.Red); // ✅

// String enums don't have this problem:
enum StrColor {
  Red = "red",
  Green = "green",
  Blue = "blue",
}
function paintWall2(color: StrColor): void {}
// paintWall2('red'); // ❌ string not assignable to StrColor — safe!

// PITFALL 2: Enums generate runtime code
enum Status {
  Active = "active",
  Inactive = "inactive",
}
// Compiles to:
// var Status;
// (function (Status) {
//   Status["Active"] = "active";
//   Status["Inactive"] = "inactive";
// })(Status || (Status = {}));
// This runtime code is in your bundle even if you only use Status.Active once

// PITFALL 3: Enum members are not the same as their values in type checking
type ActiveStatus = Status.Active; // 'active' literal type
const s: ActiveStatus = Status.Active; // ✅
const s2: ActiveStatus = "active"; // ❌ even though the VALUE is 'active'!
// String enums create branded types — 'active' !== Status.Active to TypeScript

// PITFALL 4: Ambient enums and declaration files
// const enums in .d.ts files can cause issues with isolatedModules (esbuild, swc)
```

---

## 5. Literal Types

```typescript
// A literal type is a type that represents exactly one value
type Yes = "yes";
type No = "no";
type One = 1;
type True = true;

// Literal union: a type that can be one of several specific values
type YesOrNo = "yes" | "no";
type Direction = "north" | "south" | "east" | "west";
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type Size = "xs" | "sm" | "md" | "lg" | "xl" | "2xl";

// Function accepting literal union
function request(url: string, method: HttpMethod): void {
  /* ... */
}
request("/api/users", "GET"); // ✅
request("/api/users", "FETCH"); // ❌ 'FETCH' not in HttpMethod

// Widening: TypeScript widens literals to their base type by default
let x = "north"; // TypeScript infers: string (not 'north')
let y = "north" as const; // TypeScript infers: 'north'
const z = "north"; // TypeScript infers: 'north' (const doesn't allow reassignment)

// Prevent widening with `as const` or explicit type annotation
function move(direction: Direction) {
  /* ... */
}
let dir = "north"; // string
// move(dir);       // ❌ string not assignable to Direction
move("north"); // ✅ literal is narrow in expression context
const dir2 = "north"; // 'north'
move(dir2); // ✅ const prevents widening
```

---

## 6. as const — The Modern Alternative

```typescript
// `as const` freezes values to their most literal types
const config = {
  host: "localhost",
  port: 3000,
  debug: false,
} as const;

// Without as const: { host: string; port: number; debug: boolean }
// With    as const: { readonly host: 'localhost'; readonly port: 3000; readonly debug: false }

config.host; // type: 'localhost' (literal, not string)
config.port; // type: 3000 (literal, not number)
// config.host = 'remote'; // ❌ readonly

// Most important use: deriving a union type from an array
const DIRECTIONS = ["north", "south", "east", "west"] as const;
// type: readonly ['north', 'south', 'east', 'west']
type Direction = (typeof DIRECTIONS)[number]; // 'north' | 'south' | 'east' | 'west'

const HTTP_METHODS = ["GET", "POST", "PUT", "DELETE"] as const;
type HttpMethod = (typeof HTTP_METHODS)[number]; // 'GET' | 'POST' | 'PUT' | 'DELETE'

// Single source of truth: the array is also usable at runtime
function isHttpMethod(s: string): s is HttpMethod {
  return (HTTP_METHODS as readonly string[]).includes(s);
}

// Object as const for a mapping table
const COLORS = {
  red: "#ef4444",
  green: "#22c55e",
  blue: "#3b82f6",
} as const;
type ColorName = keyof typeof COLORS; // 'red' | 'green' | 'blue'
type ColorValue = (typeof COLORS)[ColorName]; // '#ef4444' | '#22c55e' | '#3b82f6'
```

---

## 7. Discriminated Unions with Literals

```typescript
// Literal types as discriminants (covered in depth in 05-union-and-intersection-types.md)
type LoadingState = { status: "loading" };
type SuccessState<T> = { status: "success"; data: T };
type ErrorState = { status: "error"; error: Error };
type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState;

// Status literals make narrowing precise and exhaustive
function render<T>(state: AsyncState<T>): string {
  switch (state.status) {
    case "loading":
      return "...";
    case "success":
      return JSON.stringify(state.data); // data: T ✅
    case "error":
      return state.error.message; // error: Error ✅
  }
}

// Numeric discriminants also work
type Packet =
  | { kind: 0; payload: Uint8Array } // binary
  | { kind: 1; payload: string } // text
  | { kind: 2 }; // ping (no payload)
```

---

## 8. Template Literal Types

```typescript
// Combine string literals into new string literal types
type Greeting = `Hello, ${string}!`; // matches any "Hello, X!" string

// More useful: combining literal unions
type Color = "red" | "green" | "blue";
type Size = "sm" | "md" | "lg";
type ColorSize = `${Color}-${Size}`;
// 'red-sm' | 'red-md' | 'red-lg' | 'green-sm' | ... (9 combinations)

// Event names
type EventName = "click" | "focus" | "blur" | "change";
type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur' | 'onChange'

// CSS property extraction
type Padding = `padding${"Top" | "Right" | "Bottom" | "Left"}`;
// 'paddingTop' | 'paddingRight' | 'paddingBottom' | 'paddingLeft'

// Key transformation
type Getters<T> = {
  [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K];
};
type User = { name: string; age: number };
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

---

## 9. Enum vs Literal Union — When to Use Each

```
PREFER LITERAL UNIONS (as const or type alias) WHEN:
  ✅ Values are strings (most common case)
  ✅ You want tree-shakeable, zero-runtime-overhead types
  ✅ You're using esbuild, swc, or isolatedModules: true (const enum issues)
  ✅ You need to infer union types from arrays
  ✅ TypeScript-only codebase (no JS consumers)

PREFER STRING ENUMS WHEN:
  ✅ You want IDE grouping (all Status.X values grouped under Status)
  ✅ You want runtime iteration over all values (Object.values(Status))
  ✅ You want intentional "branding" — 'active' !== Status.Active by design
  ✅ Sharing types with JavaScript consumers via compiled output

PREFER NUMERIC ENUMS (or const enum) WHEN:
  ✅ Values are bit flags / masks
  ✅ Performance-critical numeric constants in a closed codebase
  ✅ Interoperating with C/C++/WASM that uses integer flags

GENERAL RULE: prefer `as const` arrays for new code — same semantics as
  string enums, zero runtime overhead, better interop, simpler
```

---

## 10. Common Mistakes

### Mistake 1 — Numeric enum type safety issue

```typescript
// ❌ Any number satisfies a numeric enum — not type-safe
enum Level {
  Low = 1,
  Mid = 2,
  High = 3,
}
function setLevel(level: Level) {}
setLevel(9999); // No error! ← use string enums or literal unions to avoid this
```

### Mistake 2 — Forgetting string enums aren't assignable from string literals

```typescript
enum Status {
  Active = "active",
}
const s: Status = "active"; // ❌ Error: string not assignable to Status
const s: Status = Status.Active; // ✅

// This is actually the DESIRED behavior for branding —
// but it surprises people who expected 'active' to just work
```

### Mistake 3 — Using `as const` on a mutable variable

```typescript
// ❌ as const on a let variable: reassignment removes the const benefit
let status = "active" as const; // type: 'active'
status = "inactive"; // ❌ Error: 'inactive' not assignable to 'active'
// So don't use as const on lets you plan to reassign

// ✅ as const is for data you won't reassign
const STATUSES = ["active", "inactive", "pending"] as const;
```

---

## 11. Exercises

### Exercise 1 — Refactor enum to as const

```typescript
// Refactor this enum to use as const + type alias
// The rest of the codebase should continue to work unchanged
enum UserRole {
  Admin = "admin",
  Editor = "editor",
  Viewer = "viewer",
}
function canEdit(role: UserRole): boolean {
  return role === UserRole.Admin || role === UserRole.Editor;
}
```

<details>
<summary>Solution</summary>

```typescript
const UserRole = {
  Admin: "admin",
  Editor: "editor",
  Viewer: "viewer",
} as const;
type UserRole = (typeof UserRole)[keyof typeof UserRole];
// UserRole is now 'admin' | 'editor' | 'viewer'

// The rest of the code works UNCHANGED because:
// UserRole.Admin is still 'admin'
// UserRole is still usable as a value (Object.values, iteration)
// The type UserRole still represents the valid values

function canEdit(role: UserRole): boolean {
  return role === UserRole.Admin || role === UserRole.Editor;
}
canEdit(UserRole.Admin); // ✅
canEdit("admin"); // ✅ (literal assignable to union — unlike string enum!)
```

</details>

---

## 🔗 Related Topics

- [`05-union-and-intersection-types.md`](./05-union-and-intersection-types.md) — Discriminated unions
- [`13-template-literal-types.md`](./13-template-literal-types.md) — Advanced template literal patterns
