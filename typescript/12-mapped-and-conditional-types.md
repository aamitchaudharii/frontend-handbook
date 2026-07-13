# 12 — Mapped and Conditional Types

> **"Mapped types iterate over the keys of a type and transform them. Conditional types branch on type-level predicates. Together they form the foundation of every utility type in TypeScript's standard library — and the foundation of every library that derives types from user schemas automatically. Once you can write your own Partial or Required, you can write anything."**

🟠 **Level: Advanced**

---

## 📚 Table of Contents

1. [Mapped Types — The Basics](#1-mapped-types--the-basics)
2. [Modifiers — readonly and optional](#2-modifiers--readonly-and-optional)
3. [Key Remapping with as](#3-key-remapping-with-as)
4. [Filtering Keys with never](#4-filtering-keys-with-never)
5. [Conditional Types Recap](#5-conditional-types-recap)
6. [Distributive Conditional Types](#6-distributive-conditional-types)
7. [Combining Mapped and Conditional Types](#7-combining-mapped-and-conditional-types)
8. [Building Standard Utility Types from Scratch](#8-building-standard-utility-types-from-scratch)
9. [Real-World Patterns](#9-real-world-patterns)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. Mapped Types — The Basics

```typescript
// Syntax: { [K in keyof T]: Transform }
// Iterates over every key K in T and defines a new type for each

// Identity (no transformation)
type Identity<T> = { [K in keyof T]: T[K] };

// Make all values strings
type Stringify<T> = { [K in keyof T]: string };
type StringUser = Stringify<{ id: number; name: string; active: boolean }>;
// { id: string; name: string; active: string }

// keyof produces the union of all keys
type User = { id: number; name: string; email: string };
type UserKey = keyof User; // 'id' | 'name' | 'email'

// [K in 'a' | 'b' | 'c']: use any union, not just keyof
type ABC = { [K in "a" | "b" | "c"]: number };
// { a: number; b: number; c: number }

// Record<K, V> is just a mapped type:
type Record_<K extends string | number | symbol, V> = { [P in K]: V };

// Map over a union to create an object type
type EventHandlers<E extends string> = {
  [K in E as `on${Capitalize<K>}`]: (event: Event) => void;
};
type Handlers = EventHandlers<"click" | "focus" | "blur">;
// { onClick: (event: Event) => void; onFocus: ...; onBlur: ... }
```

---

## 2. Modifiers — readonly and optional

```typescript
// Add modifiers with `readonly` and `?`
type Readonly_<T> = { readonly [K in keyof T]: T[K] };
type Partial_<T> = { [K in keyof T]?: T[K] };
type Required_<T> = { [K in keyof T]-?: T[K] }; // -? removes optional

// Remove modifiers with the `-` prefix:
type Mutable<T> = { -readonly [K in keyof T]: T[K] }; // remove readonly
type Required2<T> = { [K in keyof T]-?: T[K] }; // remove optional

// Combine modifier changes
type WritablePartial<T> = { -readonly [K in keyof T]?: T[K] };
type ReadonlyRequired<T> = { readonly [K in keyof T]-?: T[K] };

// Practical: frozen config type
type Config = { host?: string; port?: number };
type FinalConfig = ReadonlyRequired<Config>;
// { readonly host: string; readonly port: number }

// Deep variants
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};
type DeepRequired<T> = {
  [K in keyof T]-?: T[K] extends object ? DeepRequired<T[K]> : T[K];
};
```

---

## 3. Key Remapping with as

```typescript
// `as NewKey` in a mapped type remaps the output key
// Syntax: { [K in keyof T as NewKey]: T[K] }

// Rename all keys with a prefix
type Prefixed<T, P extends string> = {
  [K in keyof T as `${P}${Capitalize<string & K>}`]: T[K];
};
type PrefixedUser = Prefixed<{ name: string; age: number }, "get">;
// { getName: string; getAge: number }

// Generate getter methods
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number }

// Generate setter methods
type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

// Combined API type
type BeanAPI<T> = Getters<T> & Setters<T>;

// Remap to camelCase (example)
type CamelCase<S extends string> = S extends `${infer Head}_${infer Tail}`
  ? `${Head}${Capitalize<CamelCase<Tail>>}`
  : S;

type SnakeToCamel<T> = {
  [K in keyof T as CamelCase<string & K>]: T[K];
};
type DbUser = { first_name: string; last_name: string; user_id: number };
type ApiUser = SnakeToCamel<DbUser>;
// { firstName: string; lastName: string; userId: number }
```

---

## 4. Filtering Keys with never

```typescript
// In a mapped type, keys that remap to `never` are REMOVED from the output

// Keep only string-valued properties
type StringProps<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
type User = { id: number; name: string; email: string; active: boolean };
type StringUser = StringProps<User>; // { name: string; email: string }

// Keep only optional properties
type OptionalKeys<T> = {
  [K in keyof T as undefined extends T[K] ? K : never]: T[K];
};

// Keep only required properties
type RequiredKeys<T> = {
  [K in keyof T as undefined extends T[K] ? never : K]: T[K];
};

// Remove methods (keep only data properties)
type DataOnly<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};

class Service {
  name = "MyService";
  version = "1.0.0";
  private secret = "hidden";
  start() {
    /* ... */
  }
  stop() {
    /* ... */
  }
}
type ServiceData = DataOnly<Service>;
// { name: string; version: string } (methods removed)

// Nested key extraction (all leaf key paths)
type Leaves<T, P extends string = ""> = {
  [K in keyof T & string]: T[K] extends object
    ? Leaves<T[K], `${P}${K}.`>
    : `${P}${K}`;
}[keyof T & string];

type Config = { db: { host: string; port: number }; debug: boolean };
type ConfigPaths = Leaves<Config>; // 'db.host' | 'db.port' | 'debug'
```

---

## 5. Conditional Types Recap

```typescript
// T extends U ? TrueType : FalseType

// Basic uses
type IsArray<T> = T extends any[] ? true : false;
type IsPromise<T> = T extends Promise<any> ? true : false;
type IsFunction<T> = T extends Function ? true : false;

// Never as filter
type ExcludeNever<T> = T extends never ? never : T; // (identity for non-never)
// More useful: as the transform:
type FilterByType<T, U> = T extends U ? T : never;

// Nested conditional
type TypeName<T> = T extends string
  ? "string"
  : T extends number
    ? "number"
    : T extends boolean
      ? "boolean"
      : T extends null
        ? "null"
        : T extends undefined
          ? "undefined"
          : T extends Function
            ? "function"
            : "object";

type N1 = TypeName<string>; // 'string'
type N2 = TypeName<() => void>; // 'function'
type N3 = TypeName<{ a: 1 }>; // 'object'
```

---

## 6. Distributive Conditional Types

```typescript
// When the checked type is a BARE TYPE PARAMETER, TS distributes over unions:
type ToArray<T> = T extends any ? T[] : never;
type TA = ToArray<string | number>; // string[] | number[]

// Prevent distribution by wrapping in tuples:
type ToArrayNoDistrib<T> = [T] extends [any] ? T[] : never;
type TA2 = ToArrayNoDistrib<string | number>; // (string | number)[]

// Distribution makes `Exclude` and `Extract` work:
type Exclude_<T, U> = T extends U ? never : T;
type E = Exclude_<"a" | "b" | "c", "a">; // 'b' | 'c'
// Distributed: ('a' extends 'a' ? never : 'a') | ('b' extends 'a' ? never : 'b') | ...
// = never | 'b' | 'c' = 'b' | 'c'

// Filter union to only those assignable to a type
type OnlyNumbers = Extract<string | number | boolean, number>; // number

// Use distribution to apply a transform to each union member
type Boxed<T> = T extends any ? { value: T } : never;
type BoxedUnion = Boxed<string | number>; // { value: string } | { value: number }
```

---

## 7. Combining Mapped and Conditional Types

```typescript
// The real power: use conditional types as the VALUE in a mapped type

// Pick only properties whose type satisfies a condition
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};

type User = { id: number; name: string; email: string; admin: boolean };
type StringFields = PickByValue<User, string>; // { name: string; email: string }
type BooleanFields = PickByValue<User, boolean>; // { admin: boolean }

// Transform values conditionally
type Serialize<T> = {
  [K in keyof T]: T[K] extends Date ? string : T[K];
};
type RawData = { name: string; createdAt: Date; updatedAt: Date };
type JsonData = Serialize<RawData>;
// { name: string; createdAt: string; updatedAt: string }

// Nullify specific fields
type NullifyOptional<T> = {
  [K in keyof T]: undefined extends T[K] ? T[K] | null : T[K];
};

// Recursive mapped + conditional
type Simplify<T> = {
  [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : Simplify<T[K]>
    : T[K];
} extends infer O
  ? { [K in keyof O]: O[K] }
  : never;
// Collapses intersections and makes the type display cleanly in IDE hover
```

---

## 8. Building Standard Utility Types from Scratch

```typescript
// Reconstruct all built-in utility types to understand them deeply:

type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyRequired<T> = { [K in keyof T]-?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type MyRecord<K extends keyof any, V> = { [P in K]: V };
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyOmit<T, K extends keyof any> = { [P in Exclude<keyof T, K>]: T[P] };
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type MyNonNullable<T> = T extends null | undefined ? never : T;
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T;

// Verify they match the built-ins:
type Test1 = MyPartial<{ a: string; b: number }>;
// { a?: string; b?: number } ✅

type Test2 = MyReturnType<(x: string) => number>;
// number ✅
```

---

## 9. Real-World Patterns

```typescript
// PATTERN 1: API client with typed routes
type Routes = {
  "/users": { GET: User[]; POST: User };
  "/users/:id": { GET: User; PUT: User; DELETE: void };
};

type ApiClient = {
  [Route in keyof Routes]: {
    [Method in keyof Routes[Route]]: (
      ...args: Route extends `${string}:${string}` ? [{ id: string }] : []
    ) => Promise<Routes[Route][Method]>;
  };
};

// PATTERN 2: Typed form state
type FormState<T> = {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
  dirty: boolean;
  valid: boolean;
};

// PATTERN 3: Selector factory (Redux-like)
type Selector<State, Result> = (state: State) => Result;
type Selectors<State, T extends object> = {
  [K in keyof T]: Selector<State, T[K]>;
};

function createSelectors<State, T extends object>(
  selectors: Selectors<State, T>,
): Selectors<State, T> {
  return selectors;
}

// PATTERN 4: Middleware pipeline with type transformation
type Middleware<Input, Output = Input> = (input: Input) => Output;
type Pipeline<Input, Output> = Middleware<Input, Output>;

type Chain<A, B, C> = Pipeline<A, C>;
```

---

## 10. Common Mistakes

### Mistake 1 — keyof on a union type

```typescript
// keyof (A | B) = common keys of A and B (not the union of keys)
type A = { x: string; y: number };
type B = { x: string; z: boolean };
type K = keyof (A | B); // 'x' — only the key present in BOTH

// For all keys: keyof A | keyof B
type AllKeys = keyof A | keyof B; // 'x' | 'y' | 'z'
```

### Mistake 2 — Mapped type over string index signature

```typescript
type Dict = { [key: string]: number };
type DictKeys = keyof Dict; // string | number (not just string!)
// JavaScript object keys can be strings OR numbers (numbers get coerced to strings)
```

### Mistake 3 — Forgetting `as` clause needs to stay as the key type

```typescript
// ❌ as clause must produce a valid key type (string | number | symbol | never)
type Bad<T> = {
  [K in keyof T as T[K]]: K; // ❌ T[K] could be anything, not necessarily a valid key
};

// ✅ Constrain or narrow the key:
type Good<T> = {
  [K in keyof T as string & T[K]]: K; // only when T[K] is a string
};
```

---

## 11. Exercises

### Exercise 1 — Build `DeepMutable`

```typescript
// Implement DeepMutable<T> that recursively removes all `readonly` modifiers.
// DeepMutable<{ readonly x: { readonly y: string } }>
// should give { x: { y: string } }
```

<details>
<summary>Solution</summary>

```typescript
type DeepMutable<T> = {
  -readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K] // leave functions unchanged
      : DeepMutable<T[K]>
    : T[K];
};

// Test:
type Frozen = { readonly a: string; readonly b: { readonly c: number } };
type Thawed = DeepMutable<Frozen>;
// { a: string; b: { c: number } }
```

</details>

### Exercise 2 — Typed event system

```typescript
// Given a type EventMap = { [eventName: string]: unknown },
// build EventSystem<T extends EventMap> with:
// - on<K extends keyof T>(event: K, handler: (data: T[K]) => void): void
// - emit<K extends keyof T>(event: K, data: T[K]): void
// - off<K extends keyof T>(event: K, handler: (data: T[K]) => void): void
```

<details>
<summary>Solution</summary>

```typescript
type EventMap = Record<string, unknown>;

class EventSystem<T extends EventMap> {
  private handlers = new Map<keyof T, Set<(data: any) => void>>();

  on<K extends keyof T>(event: K, handler: (data: T[K]) => void): void {
    const set = this.handlers.get(event) ?? new Set();
    set.add(handler);
    this.handlers.set(event, set);
  }

  off<K extends keyof T>(event: K, handler: (data: T[K]) => void): void {
    this.handlers.get(event)?.delete(handler);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.handlers.get(event)?.forEach((h) => h(data));
  }
}

interface MyEvents {
  login: { userId: string };
  logout: { userId: string; reason?: string };
  error: { message: string; code: number };
}
const system = new EventSystem<MyEvents>();
system.on("login", ({ userId }) => console.log(userId)); // userId: string ✅
```

</details>

---

## 🔗 Related Topics

- [`11-advanced-generics.md`](./11-advanced-generics.md) — `infer` and conditional type foundations
- [`08-utility-types.md`](./08-utility-types.md) — The built-in utilities this file explains
- [`13-template-literal-types.md`](./13-template-literal-types.md) — Key remapping with template literals
