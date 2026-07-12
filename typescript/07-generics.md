# 07 — Generics

> **"Generics are how you write code that works for many types while still being type-safe. Without them, you choose between writing the same function ten times for ten different types, or abandoning type safety with `any`. Generics give you the third option: write it once, type it precisely."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [Why Generics?](#1-why-generics)
2. [Generic Functions](#2-generic-functions)
3. [Generic Interfaces and Type Aliases](#3-generic-interfaces-and-type-aliases)
4. [Generic Classes](#4-generic-classes)
5. [Constraints (extends)](#5-constraints-extends)
6. [Default Type Parameters](#6-default-type-parameters)
7. [Multiple Type Parameters](#7-multiple-type-parameters)
8. [keyof and Indexed Access](#8-keyof-and-indexed-access)
9. [Common Generic Patterns](#9-common-generic-patterns)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. Why Generics?

```typescript
// WITHOUT generics: code duplication or loss of type safety
function firstNumber(arr: number[]): number | undefined {
  return arr[0];
}
function firstString(arr: string[]): string | undefined {
  return arr[0];
}
// Repeated for every type...

// Using `any`: no type safety
function first(arr: any[]): any {
  return arr[0];
}
const x = first([1, 2, 3]);
x.toUpperCase(); // ❌ no error — but crashes at runtime!

// WITH generics: one function, full type safety
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const num = first([1, 2, 3]); // T inferred as number → returns number | undefined
const str = first(["a", "b", "c"]); // T inferred as string → returns string | undefined
const user = first(users); // T inferred as User   → returns User | undefined

num?.toFixed(2); // ✅ TypeScript knows it's number
str?.toUpperCase(); // ✅ TypeScript knows it's string
```

---

## 2. Generic Functions

```typescript
// Type parameter T is declared in angle brackets <T>
function identity<T>(value: T): T {
  return value;
}

// Multiple type parameters
function pair<A, B>(a: A, b: B): [A, B] {
  return [a, b];
}
pair("hello", 42); // [string, number]

// Explicit vs inferred type arguments
identity<string>("hello"); // explicit
identity("hello"); // inferred (preferred when unambiguous)

// Generic arrow function (TSX files need trailing comma to avoid JSX ambiguity)
const identity2 = <T>(value: T): T => value;
// or use function declaration style in TSX files

// Swap two elements in a tuple
function swap<A, B>(pair: [A, B]): [B, A] {
  return [pair[1], pair[0]];
}
swap(["hello", 42]); // [42, 'hello'] typed as [number, string]

// Map over an array with type transformation
function mapArray<T, U>(arr: T[], fn: (item: T, index: number) => U): U[] {
  return arr.map(fn);
}
const lengths = mapArray(["hello", "world"], (s) => s.length); // number[]

// Async generic function
async function fetchJSON<T>(url: string): Promise<T> {
  const res = await fetch(url);
  return res.json() as Promise<T>;
}
const user = await fetchJSON<User>("/api/user/1");
user.name; // ✅ TypeScript knows it's User
```

---

## 3. Generic Interfaces and Type Aliases

```typescript
// Generic interface
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, "id">): Promise<T>;
  update(id: string, data: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}

// Implement for a specific type
class UserRepository implements Repository<User> {
  async findById(id: string) {
    return db.users.findOne({ id });
  }
  async findAll() {
    return db.users.find();
  }
  async create(data) {
    return db.users.create(data);
  }
  async update(id, data) {
    return db.users.update({ id }, data);
  }
  async delete(id) {
    await db.users.delete({ id });
  }
}

// Generic type alias
type Nullable<T> = T | null;
type Maybe<T> = T | null | undefined;
type Awaited_<T> = T extends Promise<infer U> ? U : T; // (simplified)

type Paginated<T> = {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
};

type ApiResponse<T> =
  | { ok: true; data: T }
  | { ok: false; error: string; code: number };

// Generic with default
type WithId<T, IdType = string> = T & { id: IdType };
type UserWithId = WithId<{ name: string }>; // { name: string; id: string }
type UserWithNumId = WithId<{ name: string }, number>; // { name: string; id: number }
```

---

## 4. Generic Classes

```typescript
// Generic stack
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }
  pop(): T | undefined {
    return this.items.pop();
  }
  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }
  isEmpty(): boolean {
    return this.items.length === 0;
  }
  get size(): number {
    return this.items.length;
  }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
numStack.pop(); // number | undefined

const strStack = new Stack<string>();
strStack.push("hello");
// strStack.push(42); // ❌ Argument of type 'number' not assignable to 'string'

// Generic event emitter
class TypedEventEmitter<Events extends Record<string, unknown>> {
  private listeners = new Map<keyof Events, Set<Function>>();

  on<K extends keyof Events>(
    event: K,
    listener: (data: Events[K]) => void,
  ): void {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(listener);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners.get(event)?.forEach((l) => l(data));
  }
}

interface AppEvents {
  "user:login": { userId: string };
  "user:logout": { userId: string };
  error: { message: string; code: number };
}

const emitter = new TypedEventEmitter<AppEvents>();
emitter.on("user:login", ({ userId }) => console.log(userId)); // ✅
emitter.emit("user:login", { userId: "123" }); // ✅
// emitter.emit('user:login', { message: 'hi' }); // ❌ wrong payload type
```

---

## 5. Constraints (extends)

```typescript
// Constrain T to only types that have certain properties
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}
getLength("hello"); // ✅ strings have .length
getLength([1, 2, 3]); // ✅ arrays have .length
getLength({ length: 5, data: "x" }); // ✅ has .length
// getLength(42);    // ❌ numbers don't have .length

// Constrain to a union of types
function clamp<T extends number | bigint>(value: T, min: T, max: T): T {
  if (value < min) return min;
  if (value > max) return max;
  return value;
}

// keyof constraint: K must be a key of T
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const user = { name: "Alice", age: 30 };
getProperty(user, "name"); // string
getProperty(user, "age"); // number
// getProperty(user, 'email'); // ❌ not a key of user

// Object constraint
function merge<T extends object, U extends object>(
  target: T,
  source: U,
): T & U {
  return { ...target, ...source };
}

// Constructor constraint
function createInstance<T>(ctor: new (...args: any[]) => T): T {
  return new ctor();
}

// Abstract class constraint
function createAndLog<T extends { toString(): string }>(ctor: new () => T): T {
  const instance = new ctor();
  console.log(instance.toString());
  return instance;
}
```

---

## 6. Default Type Parameters

```typescript
// Provide a default type when the caller doesn't specify
interface Container<T = unknown> {
  value: T;
  transform<U = T>(fn: (val: T) => U): Container<U>;
}

// Common: API response with optional error type
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

// No error type specified → uses Error
const r1: Result<User> = { ok: true, value: user };

// Custom error type
const r2: Result<User, HttpError> = { ok: false, error: new HttpError(404) };

// Default in functions (less common — TS usually infers)
function wrap<T = unknown>(value: T): { wrapped: T } {
  return { wrapped: value };
}
wrap("hello"); // { wrapped: string }
wrap<number | null>(42); // { wrapped: number | null }

// Practical: paginated response with default item type
interface PaginatedResponse<Item = Record<string, unknown>> {
  items: Item[];
  total: number;
  page: number;
  pageSize: number;
}
```

---

## 7. Multiple Type Parameters

```typescript
// Two type parameters with a relationship
function zip<A, B>(as: A[], bs: B[]): [A, B][] {
  return as.map((a, i) => [a, bs[i]] as [A, B]);
}
zip(["a", "b", "c"], [1, 2, 3]); // [['a',1],['b',2],['c',3]]

// Map transformation: input type → output type
function transform<Input, Output>(
  items: Input[],
  mapper: (item: Input, index: number) => Output,
  filter?: (item: Output) => boolean,
): Output[] {
  const mapped = items.map(mapper);
  return filter ? mapped.filter(filter) : mapped;
}

// Three type parameters
function groupBy<T, K extends string | number | symbol, V = T>(
  items: T[],
  keyFn: (item: T) => K,
  valueFn?: (item: T) => V,
): Record<K, V[]> {
  const result = {} as Record<K, V[]>;
  for (const item of items) {
    const key = keyFn(item);
    const value = valueFn ? valueFn(item) : (item as unknown as V);
    (result[key] ??= []).push(value);
  }
  return result;
}

const byDept = groupBy(
  employees,
  (e) => e.department,
  (e) => e.name,
);
// Record<string, string[]> — names grouped by department
```

---

## 8. keyof and Indexed Access

```typescript
// keyof: produces a union of all property names
type User = { id: number; name: string; email: string };
type UserKey = keyof User; // 'id' | 'name' | 'email'

// Indexed access type: T[K] gives the type of T's property K
type IdType = User["id"]; // number
type NameType = User["name"]; // string

// Indexed access with union key:
type StringProps = User["name" | "email"]; // string

// Indexed access on arrays:
type FirstUser = User[][0]; // User (type of element at index 0)
const users = [{ id: 1, name: "Alice" }] as const;
type First = (typeof users)[0]; // { readonly id: 1; readonly name: 'Alice' }

// Practical: pick-by-value-type utility
type StringKeys<T> = {
  [K in keyof T]: T[K] extends string ? K : never;
}[keyof T];

type UserStringKeys = StringKeys<User>; // 'name' | 'email' (not 'id')

// Generic property getter
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}
pluck(users, "name"); // string[]
pluck(users, "id"); // number[]
// pluck(users, 'foo'); // ❌ 'foo' is not a key of User
```

---

## 9. Common Generic Patterns

```typescript
// Pattern 1: Builder with generic accumulation
class Builder<T extends object = {}> {
  private data: T;
  constructor(initial: T = {} as T) {
    this.data = initial;
  }

  set<K extends string, V>(key: K, value: V): Builder<T & Record<K, V>> {
    return new Builder({ ...this.data, [key]: value } as T & Record<K, V>);
  }

  build(): T {
    return this.data;
  }
}

const result = new Builder()
  .set("name", "Alice") // Builder<{ name: string }>
  .set("age", 30) // Builder<{ name: string; age: number }>
  .build(); // { name: string; age: number }

// Pattern 2: Pipe / compose with type tracking
function pipe<A>(value: A): A;
function pipe<A, B>(value: A, f1: (a: A) => B): B;
function pipe<A, B, C>(value: A, f1: (a: A) => B, f2: (b: B) => C): C;
function pipe(value: unknown, ...fns: Array<(x: unknown) => unknown>) {
  return fns.reduce((v, f) => f(v), value);
}

const result2 = pipe(
  "  hello world  ",
  (s: string) => s.trim(),
  (s: string) => s.split(" "),
  (words: string[]) => words.length,
); // result2: number

// Pattern 3: Recursive generic types
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

---

## 10. Common Mistakes

### Mistake 1 — Over-constraining with explicit type arguments

```typescript
// ❌ Forcing T to be a specific type defeats the purpose
function wrap<T extends string>(value: T): { value: T } {
  return { value };
}
// Now wrap only works for strings — why not just type the parameter?

// ✅ Let inference work
function wrap<T>(value: T): { value: T } {
  return { value };
}
```

### Mistake 2 — Generic where a union is simpler

```typescript
// ❌ Unnecessary generic
function process<T extends string | number>(value: T): string {
  return String(value);
}

// ✅ Just use the union
function process(value: string | number): string {
  return String(value);
}
// Use generics when the OUTPUT type needs to DEPEND ON the input type
```

### Mistake 3 — Forgetting constraints in generic functions

```typescript
// ❌ TypeScript doesn't know T has a .name property
function greet<T>(obj: T): string {
  return `Hello, ${obj.name}`; // ❌ Property 'name' does not exist on type 'T'
}

// ✅ Add a constraint
function greet<T extends { name: string }>(obj: T): string {
  return `Hello, ${obj.name}`; // ✅
}
```

---

## 11. Exercises

### Exercise 1 — Generic cache

```typescript
// Implement a generic Cache<K, V> class with:
// - get(key: K): V | undefined
// - set(key: K, value: V, ttlMs?: number): void  (optional TTL)
// - has(key: K): boolean
// - delete(key: K): void
// - clear(): void
```

<details>
<summary>Solution</summary>

```typescript
class Cache<K, V> {
  private store = new Map<K, { value: V; expiresAt?: number }>();

  set(key: K, value: V, ttlMs?: number): void {
    this.store.set(key, {
      value,
      expiresAt: ttlMs ? Date.now() + ttlMs : undefined,
    });
  }

  get(key: K): V | undefined {
    const entry = this.store.get(key);
    if (!entry) return undefined;
    if (entry.expiresAt && Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return undefined;
    }
    return entry.value;
  }

  has(key: K): boolean {
    return this.get(key) !== undefined;
  }
  delete(key: K): void {
    this.store.delete(key);
  }
  clear(): void {
    this.store.clear();
  }
}

const cache = new Cache<string, User>();
cache.set("user:1", { id: 1, name: "Alice" }, 60_000);
cache.get("user:1"); // User | undefined
```

</details>

---

## 🔗 Related Topics

- [`08-utility-types.md`](./08-utility-types.md) — Built-in generic utility types
- [`11-advanced-generics.md`](./11-advanced-generics.md) — `infer`, conditional types
- [`12-mapped-and-conditional-types.md`](./12-mapped-and-conditional-types.md) — Building custom utility types
