# 05 — Union and Intersection Types

> **"A union says 'this can be one of several things.' An intersection says 'this must be all of these things at once.' Together they cover most real-world type composition — modeling API responses that can succeed or fail, mixing in cross-cutting concerns like timestamps, and the discriminated union pattern that makes exhaustive switch statements type-safe."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [Union Types](#1-union-types)
2. [Working with Unions](#2-working-with-unions)
3. [Discriminated Unions](#3-discriminated-unions)
4. [Exhaustiveness Checking](#4-exhaustiveness-checking)
5. [Intersection Types](#5-intersection-types)
6. [Union vs Intersection — Mental Model](#6-union-vs-intersection--mental-model)
7. [Common Patterns](#7-common-patterns)
8. [Common Mistakes](#8-common-mistakes)
9. [Exercises](#9-exercises)

---

## 1. Union Types

```typescript
// A union type: the value can be ANY ONE of the listed types
type StringOrNumber = string | number;
type ID = string | number;
type Nullable<T> = T | null;
type Maybe<T> = T | null | undefined;

// Union of literal types (very common in TypeScript)
type Direction = "north" | "south" | "east" | "west";
type Status = "pending" | "loading" | "success" | "error";
type Size = "xs" | "sm" | "md" | "lg" | "xl";

// Union of object types
type Circle = { kind: "circle"; radius: number };
type Rectangle = { kind: "rect"; width: number; height: number };
type Shape = Circle | Rectangle;

// Only members common to ALL union members are accessible without narrowing
function printId(id: string | number) {
  id.toString(); // ✅ both string and number have toString()
  // id.toFixed(); // ❌ only number has toFixed()
  // id.toUpperCase(); // ❌ only string has toUpperCase()
}
```

---

## 2. Working with Unions

```typescript
// You must NARROW a union before accessing type-specific members
function format(value: string | number | boolean): string {
  if (typeof value === "string") return value.toUpperCase();
  if (typeof value === "number") return value.toFixed(2);
  return value ? "Yes" : "No"; // value must be boolean here
}

// Union with null/undefined (very common)
function getLength(str: string | null): number {
  if (str === null) return 0;
  return str.length; // str is narrowed to string here
}

// Shorter: optional chaining + nullish coalescing
const length = str?.length ?? 0;

// Arrays with union element types
type MixedArray = (string | number)[];
const arr: MixedArray = ["hello", 42, "world", 7];

// Filter narrows the type (but TypeScript needs help)
const strings = arr.filter((x): x is string => typeof x === "string");
// strings is string[] — not (string | number)[]
```

---

## 3. Discriminated Unions

```typescript
// A discriminated union has a COMMON LITERAL PROPERTY (the "discriminant")
// that uniquely identifies each member — this is the single most important
// TypeScript pattern for modeling state

// The discriminant property (here: 'status') must be a literal type
type ApiResponse<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function render<T>(response: ApiResponse<T>): string {
  switch (response.status) {
    case "loading":
      return "Loading...";
    case "success":
      return JSON.stringify(response.data); // .data only accessible here ✅
    case "error":
      return response.error.message; // .error only accessible here ✅
  }
}

// Another example: Redux/state machine actions
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" }
  | { type: "SET_USER"; user: User };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + action.amount };
    case "DECREMENT":
      return { ...state, count: state.count - action.amount };
    case "RESET":
      return initialState;
    case "SET_USER":
      return { ...state, user: action.user }; // user typed as User
  }
}

// Form validation errors
type FieldError =
  | { field: "email"; reason: "required" | "invalid" }
  | { field: "password"; reason: "required" | "too-short" | "too-weak" }
  | { field: "age"; reason: "required" | "underage" };

function handleError(err: FieldError) {
  if (err.field === "email" && err.reason === "invalid") {
    showEmailFormatHint(); // we know exactly what happened
  }
}
```

---

## 4. Exhaustiveness Checking

```typescript
// TypeScript can verify you've handled ALL union members
// by checking that a variable is typed as `never` in the default case

type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default:
      // If you add a new Shape variant and forget to handle it,
      // TypeScript will error here because shape is NOT never
      const _: never = shape;
      throw new Error(`Unhandled shape: ${JSON.stringify(shape)}`);
  }
}

// Helper function for cleaner exhaustiveness checks
function assertNever(x: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(x)}`);
}

function area2(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default:
      return assertNever(shape); // ❌ Error if a case is missing
  }
}
```

---

## 5. Intersection Types

```typescript
// An intersection: the value must satisfy ALL listed types simultaneously
type A = { name: string };
type B = { age: number };
type AB = A & B; // must have both name AND age

const person: AB = { name: "Alice", age: 30 }; // ✅
// const broken: AB = { name: 'Alice' };         // ❌ missing age

// Common use: mixing in cross-cutting concerns
type Timestamped = { createdAt: Date; updatedAt: Date };
type SoftDeletable = { deletedAt: Date | null };

type User = { id: number; name: string; email: string } & Timestamped;
type Post = { id: number; title: string; body: string } & Timestamped &
  SoftDeletable;

// Intersection of function types = most restrictive signature
type StringFn = (x: string) => void;
type NumberFn = (x: number) => void;
type BothFn = StringFn & NumberFn; // must accept EITHER string OR number

// Conflicting properties in intersection:
type X = { id: string };
type Y = { id: number };
type XY = X & Y; // id is string & number = never
// XY.id is impossible to satisfy — be careful with intersections of concrete types

// Safe pattern: intersection with Partial for optional overrides
type Config = {
  host: string;
  port: number;
  debug: boolean;
};
type PartialConfig = Config & Partial<{ timeout: number; retries: number }>;
```

---

## 6. Union vs Intersection — Mental Model

```
UNION (|) — "OR"
  type A | B: a value of this type is EITHER an A OR a B
  You can only use members COMMON to all variants (without narrowing)

  Set theory: A ∪ B (the value belongs to the set of A values OR B values)

  Example: string | number
    → can be 'hello' ✅ or 42 ✅ but NOT both at once

INTERSECTION (&) — "AND"
  type A & B: a value of this type is BOTH an A AND a B simultaneously
  You can use members from ALL intersected types (no narrowing needed)

  Set theory: A ∩ B (the value must belong to BOTH sets)

  Example: { name: string } & { age: number }
    → must have BOTH name AND age

CONFUSINGLY:
  For OBJECT types, union gives FEWER accessible properties (only shared ones)
  For OBJECT types, intersection gives MORE accessible properties (all combined)

  This is the opposite of what set theory intuition might suggest because
  we're thinking about PROPERTIES, not values:
  - Union of objects: fewer guaranteed properties (only the ones in ALL variants)
  - Intersection of objects: more properties (everything from EVERY type)
```

---

## 7. Common Patterns

```typescript
// Pattern 1: Result type (success or failure)
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

async function fetchUser(id: string): Promise<Result<User>> {
  try {
    const user = await api.getUser(id);
    return { ok: true, value: user };
  } catch (e) {
    return { ok: false, error: e as Error };
  }
}

const result = await fetchUser("123");
if (result.ok) {
  console.log(result.value.name); // ✅ value is User
} else {
  console.error(result.error.message); // ✅ error is Error
}

// Pattern 2: With / WithRequired utility
type WithId<T> = T & { id: string };
type WithRequired<T, K extends keyof T> = T & Required<Pick<T, K>>;

// Pattern 3: Loose vs Strict mode using union
type LooseSize = "sm" | "md" | "lg" | string; // allows any string (loose)
type StrictSize = "sm" | "md" | "lg"; // only those 3 (strict)

// Pattern 4: Payload-typed events
type AppEvent =
  | { event: "user.created"; payload: { userId: string } }
  | { event: "order.placed"; payload: { orderId: string; total: number } }
  | { event: "page.viewed"; payload: { url: string } };

function track<E extends AppEvent["event"]>(
  event: E,
  payload: Extract<AppEvent, { event: E }>["payload"],
) {
  /* ... */
}

track("user.created", { userId: "123" }); // ✅
track("order.placed", { orderId: "456", total: 99.99 }); // ✅
// track('user.created', { orderId: '456' });   // ❌ wrong payload type
```

---

## 8. Common Mistakes

### Mistake 1 — Union doesn't mean "both at once"

```typescript
type A = { x: string };
type B = { y: number };
type AorB = A | B;

const val: AorB = { x: "hello", y: 42 }; // ✅ allowed (satisfies A AND B)
// But TypeScript can only guarantee what's in ALL variants:
val.x; // ❌ Error — y might not be there (it could be just B)
val.y; // ❌ Error — x might not be there (it could be just A)

// ✅ Narrow first
if ("x" in val) {
  val.x;
} // OK — narrowed to A
if ("y" in val) {
  val.y;
} // OK — narrowed to B
```

### Mistake 2 — Missing discriminant means no narrowing

```typescript
// ❌ No discriminant — TypeScript can't narrow
type Cat = { name: string; purr: () => void };
type Dog = { name: string; bark: () => void };
type Pet = Cat | Dog;

function speak(pet: Pet) {
  pet.bark(); // ❌ bark doesn't exist on Cat
  // Can only use .name safely
}

// ✅ Add a discriminant
type Cat2 = { kind: "cat"; name: string; purr: () => void };
type Dog2 = { kind: "dog"; name: string; bark: () => void };
type Pet2 = Cat2 | Dog2;

function speak2(pet: Pet2) {
  if (pet.kind === "dog") pet.bark(); // ✅ narrowed to Dog2
}
```

---

## 9. Exercises

### Exercise 1 — Model a payment system

```typescript
// Model a payment result as a discriminated union with these variants:
// - Success: has transactionId (string) and amount (number)
// - Declined: has reason ('insufficient-funds' | 'card-expired' | 'fraud')
// - Error: has message (string) and code (number)
// Then write processPayment(result: PaymentResult): string
// that returns a user-facing message for each case.
```

<details>
<summary>Solution</summary>

```typescript
type PaymentResult =
  | { status: "success"; transactionId: string; amount: number }
  | {
      status: "declined";
      reason: "insufficient-funds" | "card-expired" | "fraud";
    }
  | { status: "error"; message: string; code: number };

function processPayment(result: PaymentResult): string {
  switch (result.status) {
    case "success":
      return `Payment of $${result.amount} confirmed (ID: ${result.transactionId})`;
    case "declined":
      const reasons = {
        "insufficient-funds": "Insufficient funds",
        "card-expired": "Card has expired",
        fraud: "Transaction flagged for review",
      };
      return `Payment declined: ${reasons[result.reason]}`;
    case "error":
      return `Payment error ${result.code}: ${result.message}`;
    default:
      const _: never = result;
      throw new Error("Unhandled payment result");
  }
}
```

</details>

---

## 🔗 Related Topics

- [`06-type-narrowing.md`](./06-type-narrowing.md) — How TypeScript narrows union types
- [`11-advanced-generics.md`](./11-advanced-generics.md) — `Extract`, `Exclude` with unions
- [`12-mapped-and-conditional-types.md`](./12-mapped-and-conditional-types.md) — Distributing over unions
