# 11 — Advanced Generics

> **"Once you understand basic generics, the next leap is conditional types and `infer` — the tools that let you write types that compute results based on their inputs. This is where TypeScript's type system starts to look like a programming language in its own right. The patterns here power every major library's type definitions: React, Zod, tRPC, Prisma."**

🟠 **Level: Advanced**

---

## 📚 Table of Contents

1. [Conditional Types](#1-conditional-types)
2. [infer — Extracting Types from Type Positions](#2-infer--extracting-types-from-type-positions)
3. [Distributive Conditional Types](#3-distributive-conditional-types)
4. [Recursive Conditional Types](#4-recursive-conditional-types)
5. [Variance — Covariance and Contravariance](#5-variance--covariance-and-contravariance)
6. [Generic Constraints Patterns](#6-generic-constraints-patterns)
7. [Type-Level Programming Patterns](#7-type-level-programming-patterns)
8. [Common Mistakes](#8-common-mistakes)
9. [Exercises](#9-exercises)

---

## 1. Conditional Types

```typescript
// Syntax: T extends U ? TypeIfTrue : TypeIfFalse
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
type C = IsString<"hello">; // true (literal extends string)

// Practical: flatten one level of array
type Flatten<T> = T extends Array<infer Item> ? Item : T;
type F1 = Flatten<string[]>; // string
type F2 = Flatten<number>; // number (not an array, returns T itself)

// Conditional with union discriminant
type IsNullable<T> = null extends T ? true : false;
type N1 = IsNullable<string | null>; // true
type N2 = IsNullable<string>; // false

// Built-in utility types ARE conditional types:
type NonNullable_<T> = T extends null | undefined ? never : T;
type Awaited_<T> = T extends Promise<infer U> ? Awaited_<U> : T;
type ReturnType_<T> = T extends (...args: any[]) => infer R ? R : never;

// Conditional types can be complex
type Diff<T, U> = T extends U ? never : T; // T minus U
type Filter<T, U> = T extends U ? T : never; // keep only T assignable to U
```

---

## 2. infer — Extracting Types from Type Positions

```typescript
// `infer X` in a conditional type: capture a type at that position as X
// Can only appear in the `extends` clause of a conditional type

// Extract element type from an array
type ElementOf<T> = T extends (infer E)[] ? E : never;
type E1 = ElementOf<string[]>; // string
type E2 = ElementOf<[1, 2, 3]>; // 1 | 2 | 3

// Extract return type of a function
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type R1 = MyReturnType<() => string>; // string
type R2 = MyReturnType<(x: number) => boolean>; // boolean

// Extract parameter types
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;
type P1 = MyParameters<(a: string, b: number) => void>; // [string, number]

// Extract first parameter
type FirstParam<T> = T extends (first: infer F, ...rest: any[]) => any
  ? F
  : never;
type FP = FirstParam<(name: string, age: number) => void>; // string

// Extract Promise value
type Unwrap<T> = T extends Promise<infer V> ? Unwrap<V> : T;
type U1 = Unwrap<Promise<Promise<string>>>; // string

// Extract from object property types
type PropType<T, K extends keyof T> = T extends { [P in K]: infer V }
  ? V
  : never;
type PT = PropType<{ name: string; age: number }, "name">; // string

// Multiple infer in one conditional
type ExtractFnArgs<T> = T extends (arg1: infer A, arg2: infer B) => infer R
  ? { args: [A, B]; returnType: R }
  : never;

type F = ExtractFnArgs<(s: string, n: number) => boolean>;
// { args: [string, number]; returnType: boolean }

// infer in tuple positions (TS 4.7+)
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type Tail<T extends any[]> = T extends [any, ...infer T] ? T : never;
type Last<T extends any[]> = T extends [...any[], infer L] ? L : never;

type H = Head<[string, number, boolean]>; // string
type T = Tail<[string, number, boolean]>; // [number, boolean]
type L = Last<[string, number, boolean]>; // boolean
```

---

## 3. Distributive Conditional Types

```typescript
// When T in `T extends U ? X : Y` is a BARE TYPE PARAMETER,
// TypeScript distributes the conditional over each union member:

type ToArray<T> = T extends any ? T[] : never;
type A = ToArray<string | number>;
// Distributes: (string extends any ? string[] : never) | (number extends any ? number[] : never)
// Result: string[] | number[]

// Compare: WITHOUT distribution (wrap in a tuple to prevent it)
type ToArrayNoDistrib<T> = [T] extends [any] ? T[] : never;
type B = ToArrayNoDistrib<string | number>; // (string | number)[]

// Practical: Filter union members
type Filter<T, U> = T extends U ? T : never;
type Exclude_<T, U> = T extends U ? never : T;

type OnlyStrings = Filter<string | number | boolean, string>; // string
type NoStrings = Exclude_<string | number | boolean, string>; // number | boolean

// Why `never` disappears from unions:
// never | string | number = string | number
// (never is the identity element for union types)

// Distributive to split a union into individual tuple branches:
type UnionToTuple<T> = /* complex — requires some workarounds */ never; // see advanced patterns

// Guard against unintended distribution:
type IsNever<T> = [T] extends [never] ? true : false; // ← brackets prevent distribution
type N1 = IsNever<never>; // true
type N2 = IsNever<string>; // false

// Without brackets:
type IsBadNever<T> = T extends never ? true : false;
type N3 = IsBadNever<never>; // never (distributed over empty union!)
```

---

## 4. Recursive Conditional Types

```typescript
// TypeScript 4.1+ supports recursive conditional types

// Deep readonly
type DeepReadonly<T> = T extends (infer E)[]
  ? ReadonlyArray<DeepReadonly<E>>
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;

type Config = { db: { host: string; ports: number[] }; debug: boolean };
type ReadonlyConfig = DeepReadonly<Config>;
// { readonly db: { readonly host: string; readonly ports: readonly number[] }; readonly debug: boolean }

// Deep partial
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Flatten nested arrays
type Flatten<T> =
  T extends Array<infer Item> ? (Item extends any[] ? Flatten<Item> : Item) : T;
type F = Flatten<string[][][]>; // string

// JSON-safe types
type JSONPrimitive = string | number | boolean | null;
type JSONValue = JSONPrimitive | JSONValue[] | { [key: string]: JSONValue };

// Path-based accessor (advanced)
type Path<T, K extends keyof T = keyof T> = K extends string
  ? T[K] extends Record<string, any>
    ? K | `${K}.${Path<T[K]>}`
    : K
  : never;

type UserPath = Path<{ name: string; address: { city: string; zip: string } }>;
// 'name' | 'address' | 'address.city' | 'address.zip'
```

---

## 5. Variance — Covariance and Contravariance

```typescript
// COVARIANCE: a generic type A<T> is covariant if T extends U implies A<T> extends A<U>
// In TypeScript: return types and read positions are covariant

type Producer<T> = () => T;
// Dog extends Animal → Producer<Dog> extends Producer<Animal>
// (a function that produces a Dog can be used where an Animal-producer is needed)

// CONTRAVARIANCE: A<T> is contravariant if T extends U implies A<U> extends A<T>
// (the direction FLIPS)
// In TypeScript: function parameter types are contravariant

type Consumer<T> = (x: T) => void;
// Dog extends Animal → Consumer<Animal> extends Consumer<Dog>
// (a function that consumes ANY Animal can be used where a Dog-consumer is needed —
// because it's safe to pass a Dog to a function that accepts any Animal)

// BIVARIANCE: T doesn't restrict the direction (both co and contra)
// TypeScript used to treat method parameters bivariantly (for practicality)
// strictFunctionTypes: true makes method signatures properly contravariant

// EXPLICIT VARIANCE ANNOTATIONS (TypeScript 4.7+)
interface Box<out T> {
  // `out` = covariant: T is only in output (read) positions
  get(): T;
  // set(x: T): void; // ❌ Error: 'out' parameter T used in input position
}

interface Writer<in T> {
  // `in` = contravariant: T is only in input (write) positions
  write(value: T): void;
  // read(): T; // ❌ Error: 'in' parameter T used in output position
}

interface Transformer<in In, out Out> {
  // invariant-like
  transform(input: In): Out;
}

// Why variance matters:
// Correct variance annotations let TypeScript catch more bugs
// and can improve type-checking performance on large codebases
```

---

## 6. Generic Constraints Patterns

```typescript
// Constraint to object with specific key
function hasKey<T extends object, K extends PropertyKey>(
  obj: T,
  key: K,
): obj is T & Record<K, unknown> {
  return key in obj;
}

// Constraint using conditional types
type StringKeyOf<T> = keyof T & string;
function getStringKey<T>(obj: T, key: StringKeyOf<T>): T[StringKeyOf<T>] {
  return obj[key];
}

// Ensure T is not null/undefined
type NonNullish<T> = T extends null | undefined ? never : T;
function process<T>(value: NonNullish<T>): void {
  /* ... */
}

// Callable constraint
type Fn = (...args: any[]) => any;
function memoize<T extends Fn>(fn: T): T {
  const cache = new Map();
  return ((...args: any[]) => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  }) as T;
}

// Constructor constraint
type Class<T = any> = new (...args: any[]) => T;
function injectable<T extends Class>(Base: T) {
  return class extends Base {
    static injected = true;
  };
}

// Constraint to same type (reflexive)
function assertEqual<T, U extends T>(expected: T, actual: U): void {
  /* ... */
}
```

---

## 7. Type-Level Programming Patterns

```typescript
// PATTERN 1: Type-safe event bus
type EventMap = Record<string, unknown>;
type Listener<T extends EventMap, K extends keyof T> = (data: T[K]) => void;

class TypedEventBus<T extends EventMap> {
  private listeners = new Map<keyof T, Set<Listener<T, any>>>();

  on<K extends keyof T>(event: K, fn: Listener<T, K>): () => void {
    const set = this.listeners.get(event) ?? new Set();
    set.add(fn);
    this.listeners.set(event, set);
    return () => set.delete(fn);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.listeners.get(event)?.forEach((fn) => fn(data));
  }
}

// Usage is fully type-safe:
interface AppEvents {
  "user:login": { userId: string; timestamp: number };
  "item:added": { itemId: string; quantity: number };
}
const bus = new TypedEventBus<AppEvents>();
bus.on("user:login", ({ userId, timestamp }) => {
  /* userId: string ✅ */
});
// bus.on('user:login', ({ orderId }) => {}); // ❌ 'orderId' doesn't exist

// PATTERN 2: Schema inference (simplified Zod-like)
type Schema<T> = {
  _type: T;
  parse(input: unknown): T;
};

function string(): Schema<string> {
  return {
    _type: "" as string,
    parse(input) {
      if (typeof input !== "string") throw new TypeError("Expected string");
      return input;
    },
  };
}
function object<T extends Record<string, Schema<any>>>(
  shape: T,
): Schema<{
  [K in keyof T]: T[K]["_type"];
}> {
  return {
    _type: {} as any,
    parse(input) {
      if (typeof input !== "object" || !input)
        throw new TypeError("Expected object");
      const result: any = {};
      for (const [key, schema] of Object.entries(shape)) {
        result[key] = schema.parse((input as any)[key]);
      }
      return result;
    },
  };
}

const UserSchema = object({ name: string(), email: string() });
type UserType = (typeof UserSchema)["_type"]; // { name: string; email: string }
```

---

## 8. Common Mistakes

### Mistake 1 — Forgetting that `infer` only works in `extends` clauses

```typescript
// ❌ infer outside a conditional type is invalid
type Bad<T> = infer U; // SyntaxError

// ✅ infer inside a conditional type
type Good<T> = T extends Array<infer U> ? U : never;
```

### Mistake 2 — Unexpected distribution over naked type parameters

```typescript
// ❌ Surprised by distribution
type Wrap<T> = T extends any ? { value: T } : never;
type W = Wrap<string | number>; // { value: string } | { value: number }
// Not { value: string | number }!

// ✅ Prevent distribution with tuple wrapping
type WrapNoDistrib<T> = [T] extends [any] ? { value: T } : never;
type W2 = WrapNoDistrib<string | number>; // { value: string | number }
```

### Mistake 3 — Recursive types causing infinite instantiation

```typescript
// ❌ TypeScript may error: "Type instantiation is excessively deep"
type InfiniteRecurse<T> = T extends object ? InfiniteRecurse<T[keyof T]> : T;

// ✅ Add a depth limit using tuple accumulation
type Recurse<T, Depth extends unknown[] = []> = Depth["length"] extends 10 // max depth of 10
  ? T
  : T extends object
    ? Recurse<T[keyof T], [...Depth, unknown]>
    : T;
```

---

## 9. Exercises

### Exercise 1 — Build `PromiseAll`

```typescript
// Implement the type for Promise.all:
// PromiseAll<[Promise<A>, Promise<B>, Promise<C>]> → Promise<[A, B, C]>
// Hint: use mapped types + infer + tuple manipulation
```

<details>
<summary>Solution</summary>

```typescript
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type AwaitAll<T extends any[]> = {
  [K in keyof T]: Awaited<T[K]>;
};

// Test:
type R = AwaitAll<[Promise<string>, Promise<number>, Promise<boolean>]>;
// [string, number, boolean]

// The real Promise.all signature in TypeScript's lib uses complex overloads,
// but this captures the core idea
```

</details>

---

## 🔗 Related Topics

- [`12-mapped-and-conditional-types.md`](./12-mapped-and-conditional-types.md) — Mapped types using these patterns
- [`08-utility-types.md`](./08-utility-types.md) — Built-in types built with these techniques
- [`javascript-core/26-iterators-and-generators.md`](../javascript-core/26-iterators-and-generators.md) — Iterator types
