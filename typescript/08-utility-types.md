# 08 — Utility Types

> **"TypeScript ships with a library of generic utility types that solve the most common type transformation problems. Knowing them by heart means you stop reinventing them — and start recognising when a complex type you're trying to write is just a composition of ones that already exist."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [Partial and Required](#1-partial-and-required)
2. [Readonly and Mutable](#2-readonly-and-mutable)
3. [Pick and Omit](#3-pick-and-omit)
4. [Record](#4-record)
5. [Extract and Exclude](#5-extract-and-exclude)
6. [NonNullable](#6-nonnullable)
7. [ReturnType, Parameters, InstanceType](#7-returntype-parameters-instancetype)
8. [Awaited](#8-awaited)
9. [ConstructorParameters](#9-constructorparameters)
10. [Template Literal Utility Types](#10-template-literal-utility-types)
11. [Composing Utility Types](#11-composing-utility-types)
12. [Common Mistakes](#12-common-mistakes)
13. [Exercises](#13-exercises)

---

## 1. Partial and Required

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  bio?: string; // already optional
}

// Partial<T>: makes ALL properties optional
type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string; bio?: string }

// USE CASE: update/patch operations
function updateUser(id: number, changes: Partial<User>): User {
  return { ...currentUser, ...changes };
}
updateUser(1, { name: "Bob" }); // ✅ only name
updateUser(1, { name: "Bob", bio: "..." }); // ✅ multiple fields
// updateUser(1, { unknown: 'x' });       // ❌

// Required<T>: makes ALL properties required (removes ?)
type RequiredUser = Required<User>;
// { id: number; name: string; email: string; bio: string }
// bio is now required even though it was optional

// USE CASE: after validation, all fields are guaranteed
function validateUser(data: Partial<User>): Required<User> {
  if (!data.id || !data.name || !data.email) throw new Error("Missing fields");
  return {
    id: data.id,
    name: data.name,
    email: data.email,
    bio: data.bio ?? "",
  };
}

// Combining: Required of Partial gives you all properties non-optional
// (same as Required<T> for a clean type — useful when building types conditionally)
```

---

## 2. Readonly and Mutable

```typescript
interface Config {
  host: string;
  port: number;
  debug: boolean;
}

// Readonly<T>: makes ALL properties readonly
type ReadonlyConfig = Readonly<Config>;
// { readonly host: string; readonly port: number; readonly debug: boolean }

const config: ReadonlyConfig = { host: "localhost", port: 3000, debug: false };
// config.port = 8080; // ❌ Cannot assign to 'port' (readonly)

// USE CASE: configuration objects, constants, Redux state
const initialState: Readonly<State> = { count: 0, users: [] };

// ReadonlyArray<T>: immutable array
const names: ReadonlyArray<string> = ["Alice", "Bob"];
// names.push('Carol'); // ❌ push doesn't exist on ReadonlyArray
// names[0] = 'X';      // ❌ index assignment not allowed

// Or: readonly T[]
const ids: readonly number[] = [1, 2, 3];

// Mutable<T>: TypeScript doesn't ship this — create your own
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
// -readonly removes the readonly modifier

type MutableConfig = Mutable<ReadonlyConfig>;
// { host: string; port: number; debug: boolean } — all writeable again

// Deep readonly (not built-in)
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

---

## 3. Pick and Omit

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
}

// Pick<T, K>: keep ONLY the listed keys
type PublicUser = Pick<User, "id" | "name" | "email">;
// { id: number; name: string; email: string }

// USE CASE: DTO / response shaping
function toPublicUser(user: User): PublicUser {
  return { id: user.id, name: user.name, email: user.email };
}

// Omit<T, K>: remove the listed keys (keep everything else)
type SafeUser = Omit<User, "password">;
// { id: number; name: string; email: string; createdAt: Date; updatedAt: Date }

// USE CASE: remove sensitive fields
function stripSensitiveData(user: User): SafeUser {
  const { password, ...safe } = user;
  return safe;
}

// Omit for create/update inputs (remove server-managed fields)
type CreateUserInput = Omit<User, "id" | "createdAt" | "updatedAt">;
type UpdateUserInput = Partial<Omit<User, "id" | "createdAt" | "updatedAt">>;

// Pick vs Omit — when to use which:
// Pick  → when you want a SMALL SUBSET of a large type (name what you want)
// Omit  → when you want MOST of a type minus a few fields (name what to remove)
// Pick(4 keys from 20) vs Omit(16 keys from 20) → Pick wins
// Pick(16 keys from 20) vs Omit(4 keys from 20) → Omit wins
```

---

## 4. Record

```typescript
// Record<K, V>: an object type with keys K and values V
type StringMap = Record<string, string>; // { [key: string]: string }
type NumberMap = Record<string, number>;
type UserMap = Record<string, User>; // keyed by string IDs

// USE CASE: lookup tables, maps, dictionaries
const httpStatusMessages: Record<number, string> = {
  200: "OK",
  404: "Not Found",
  500: "Internal Server Error",
};

// Record with a union of literal keys (more precise than Record<string,...>)
type StatusColor = Record<"active" | "inactive" | "pending", string>;
const colors: StatusColor = {
  active: "#22c55e",
  inactive: "#6b7280",
  pending: "#f59e0b",
};
// colors.unknown; // ❌ not a key of StatusColor

// Record for groupBy results
function groupBy<T, K extends string | number>(
  items: T[],
  keyFn: (item: T) => K,
): Partial<Record<K, T[]>> {
  const result: Partial<Record<K, T[]>> = {};
  for (const item of items) {
    const key = keyFn(item);
    (result[key] ??= []).push(item);
  }
  return result;
}
```

---

## 5. Extract and Exclude

```typescript
// Exclude<T, U>: removes from T the types assignable to U
type A = string | number | boolean | null;
type NonNull = Exclude<A, null>; // string | number | boolean
type OnlyStrings = Exclude<A, number | boolean | null>; // string

// Extract<T, U>: keeps from T only the types assignable to U
type NumbersOnly = Extract<A, number>; // number
type StringOrNum = Extract<A, string | number>; // string | number

// USE CASE: filter discriminated union members
type Action =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "RESET" }
  | { type: "SET_USER"; user: User };

// Extract only actions that have a `user` property
type UserActions = Extract<Action, { user: User }>; // { type: 'SET_USER'; user: User }
// Exclude actions that have a `user` property
type SimpleActions = Exclude<Action, { user: User }>; // INCREMENT | DECREMENT | RESET

// Exclude from string union
type EventNames = "click" | "focus" | "blur" | "keydown" | "keyup";
type KeyEvents = Extract<EventNames, `key${string}`>; // 'keydown' | 'keyup'
type NonKey = Exclude<EventNames, `key${string}`>; // 'click' | 'focus' | 'blur'
```

---

## 6. NonNullable

```typescript
// NonNullable<T>: removes null and undefined from T
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string

type MaybeUser = User | null | undefined;
type DefiniteUser = NonNullable<MaybeUser>; // User

// USE CASE: after null-checking, assure TypeScript the value is present
function requireValue<T>(
  value: T | null | undefined,
  name: string,
): NonNullable<T> {
  if (value == null) throw new Error(`${name} is required`);
  return value as NonNullable<T>;
}

const user = requireValue(maybeUser, "user"); // User (not User | null | undefined)

// With arrays: filter out nulls
const maybes = [1, null, 2, undefined, 3];
const defined: NonNullable<(typeof maybes)[number]>[] = maybes.filter(
  (x): x is NonNullable<typeof x> => x != null,
);
// defined: number[]
```

---

## 7. ReturnType, Parameters, InstanceType

```typescript
// ReturnType<T>: extracts the return type of a function type
function fetchUser(): Promise<User> {
  return api.getUser();
}
type FetchResult = ReturnType<typeof fetchUser>; // Promise<User>
type User2 = Awaited<ReturnType<typeof fetchUser>>; // User

// USE CASE: infer types from existing functions without duplication
function createStore() {
  return {
    count: 0,
    increment() {
      this.count++;
    },
  };
}
type StoreType = ReturnType<typeof createStore>;
// { count: number; increment(): void }

// Parameters<T>: extracts parameter types as a tuple
function connect(host: string, port: number, ssl: boolean): void {}
type ConnectParams = Parameters<typeof connect>; // [string, number, boolean]

// USE CASE: forwarding parameters, currying
function withLogging<T extends (...args: any[]) => any>(fn: T) {
  return (...args: Parameters<T>): ReturnType<T> => {
    console.log("Calling with:", args);
    return fn(...args);
  };
}

// InstanceType<T>: extracts the instance type of a constructor
class MyService {
  run(): void {}
}
type ServiceInstance = InstanceType<typeof MyService>; // MyService

// USE CASE: factory functions that take constructors
function createService<T extends new (...args: any[]) => any>(
  Svc: T,
): InstanceType<T> {
  return new Svc();
}
```

---

## 8. Awaited

```typescript
// Awaited<T>: recursively unwraps Promise types
type A = Awaited<Promise<string>>; // string
type B = Awaited<Promise<Promise<number>>>; // number (recursive)
type C = Awaited<string>; // string (non-Promise passthrough)

// USE CASE: getting the resolved type of an async function
async function getUser(): Promise<User> {
  /* ... */
}
type ResolvedUser = Awaited<ReturnType<typeof getUser>>; // User

// Replacing the old manual version:
// type Unwrap<T> = T extends Promise<infer U> ? Unwrap<U> : T; // (Awaited does this)

// Practical: get array element type from async function
async function getUsers(): Promise<User[]> {
  /* ... */
}
type UserArrayElement = Awaited<ReturnType<typeof getUsers>>[number]; // User
```

---

## 9. ConstructorParameters

```typescript
// ConstructorParameters<T>: extracts constructor parameter types as a tuple
class HttpClient {
  constructor(
    private baseUrl: string,
    private timeout: number,
    private headers?: Record<string, string>,
  ) {}
}

type HttpClientArgs = ConstructorParameters<typeof HttpClient>;
// [string, number, Record<string, string>?]

// USE CASE: factory that forwards constructor args
function createCached<T extends new (...args: any[]) => any>(
  Cls: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new Cls(...args);
}
const client = createCached(HttpClient, "https://api.example.com", 5000);
```

---

## 10. Template Literal Utility Types

```typescript
// Built-in string utility types (TypeScript 4.1+)
type Upper = Uppercase<"hello">; // 'HELLO'
type Lower = Lowercase<"HELLO">; // 'hello'
type Cap = Capitalize<"hello">; // 'Hello'
type Uncap = Uncapitalize<"Hello">; // 'hello'

// USE CASE: transform event names
type EventKey = "click" | "focus" | "blur";
type HandlerKey = `on${Capitalize<EventKey>}`; // 'onClick' | 'onFocus' | 'onBlur'

// Getters and setters
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

type User = { name: string; age: number };
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

---

## 11. Composing Utility Types

```typescript
// Utility types compose well — this is their biggest strength

// Create input type for an API endpoint
type CreateInput<T> = Omit<T, "id" | "createdAt" | "updatedAt">;
type UpdateInput<T> = Partial<Omit<T, "id" | "createdAt" | "updatedAt">>;
type PublicOutput<T> = Omit<T, "password" | "secret" | "__v">;

// Concrete types derived from User
type CreateUserInput = CreateInput<User>;
type UpdateUserInput = UpdateInput<User>;
type PublicUserOutput = PublicOutput<User>;

// Deep partial
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Require specific keys while making the rest optional
type RequireKeys<T, K extends keyof T> = T & Required<Pick<T, K>>;
type UserWithRequiredEmail = RequireKeys<Partial<User>, "email">;
// All fields optional EXCEPT email which is required
```

---

## 12. Common Mistakes

### Mistake 1 — Omit doesn't work well with union types

```typescript
type A = { id: number; name: string };
type B = { id: number; value: boolean };
type AB = A | B;

// ❌ Omit on a union distributes over BOTH types, removing 'id' from each
type NoId = Omit<AB, "id">; // { name: string } | { value: boolean }

// ✅ Use a distributive helper:
type DistributiveOmit<T, K extends keyof T> = T extends unknown
  ? Omit<T, K>
  : never;
type NoId2 = DistributiveOmit<AB, "id">; // { name: string } | { value: boolean }
// (same result here, but important for more complex cases)
```

### Mistake 2 — Partial is shallow

```typescript
// Partial only goes ONE level deep
type Config = { db: { host: string; port: number }; debug: boolean };
type P = Partial<Config>;
// { db?: { host: string; port: number }; debug?: boolean }
// db is optional BUT its properties (host, port) are still required!

// ✅ For deep partial, use a recursive utility type (Section 11)
```

---

## 13. Exercises

### Exercise 1 — Build UpdateInput

```typescript
// Given any model type T, create a type UpdateInput<T> that:
// 1. Makes all fields optional
// 2. Removes 'id', 'createdAt', 'updatedAt' fields
// 3. Requires at least one field to be present (no empty update objects)

// Bonus: the "at least one field" constraint
```

<details>
<summary>Solution</summary>

```typescript
type SystemFields = "id" | "createdAt" | "updatedAt";

// Basic: optional + no system fields
type UpdateInput<T> = Partial<Omit<T, SystemFields>>;

// With "at least one field" constraint:
type AtLeastOne<T> = {
  [K in keyof T]-?: Required<Pick<T, K>> & Partial<Omit<T, K>>;
}[keyof T];

type StrictUpdateInput<T> = AtLeastOne<Omit<T, SystemFields>>;

// Usage:
type UpdateUserInput = StrictUpdateInput<User>;
// const empty: UpdateUserInput = {}; // ❌ Error — at least one field required
// const ok: UpdateUserInput = { name: 'Bob' }; // ✅
```

</details>

---

## 🔗 Related Topics

- [`07-generics.md`](./07-generics.md) — How these utility types are built
- [`12-mapped-and-conditional-types.md`](./12-mapped-and-conditional-types.md) — Building your own utility types
