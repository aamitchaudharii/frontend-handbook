# 16 — Advanced Type Patterns

> **"Senior TypeScript isn't about knowing more syntax — it's about reaching for the right pattern when a simpler approach would compile but mislead. Branded types make invalid states unrepresentable. Opaque types prevent mixing up two strings that mean different things. The builder pattern makes incomplete objects type errors instead of runtime bugs. These patterns protect entire categories of bugs at the type level."**

🔴 **Level: Senior**

---

## 📚 Table of Contents

1. [Branded Types](#1-branded-types)
2. [Opaque Types](#2-opaque-types)
3. [The Builder Pattern with Type Accumulation](#3-the-builder-pattern-with-type-accumulation)
4. [Type-Safe Error Handling (Result Type)](#4-type-safe-error-handling-result-type)
5. [Phantom Types](#5-phantom-types)
6. [Nominal Typing in a Structural System](#6-nominal-typing-in-a-structural-system)
7. [Type-Safe Event Bus](#7-type-safe-event-bus)
8. [Exhaustive Pattern Matching](#8-exhaustive-pattern-matching)
9. [Type-Level Validation](#9-type-level-validation)
10. [Recursive Type Utilities](#10-recursive-type-utilities)
11. [Common Mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)

---

## 1. Branded Types

```typescript
// PROBLEM: Two IDs are both `string`, but mixing them up is a bug
function getUser(userId: string) {
  /* ... */
}
function getPost(postId: string) {
  /* ... */
}

const userId = "user-123";
const postId = "post-456";
getUser(postId); // ✅ TypeScript doesn't catch this — both are `string`!

// SOLUTION: Branded types — intersect with a unique marker
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;
type OrderId = Brand<string, "OrderId">;

// Create branded values via constructor functions (the only way to create them)
function createUserId(id: string): UserId {
  return id as UserId;
}
function createPostId(id: string): PostId {
  return id as PostId;
}

// Now type errors catch the mix-up:
function getUser(userId: UserId): Promise<User> {
  /* ... */
}
function getPost(postId: PostId): Promise<Post> {
  /* ... */
}

const uid = createUserId("user-123");
const pid = createPostId("post-456");

getUser(uid); // ✅
getPost(pid); // ✅
getUser(pid); // ❌ Argument of type 'PostId' not assignable to 'UserId'
getUser("user-123"); // ❌ string not assignable to UserId — must use createUserId

// Common use cases for branding:
type Dollars = Brand<number, "Dollars">; // vs Cents or Euros
type Cents = Brand<number, "Cents">;
type Latitude = Brand<number, "Latitude">; // vs Longitude
type Longitude = Brand<number, "Longitude">;
type EmailStr = Brand<string, "Email">; // validated email string
type UrlStr = Brand<string, "Url">; // validated URL string
type NonEmpty = Brand<string, "NonEmpty">; // guaranteed non-empty string

// With runtime validation
function parseEmail(input: string): EmailStr {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input)) {
    throw new Error(`Invalid email: ${input}`);
  }
  return input as EmailStr;
}

function sendEmail(to: EmailStr, subject: string): void {
  /* ... */
}
sendEmail(parseEmail("alice@example.com"), "Hello"); // ✅
// sendEmail('alice@example.com', 'Hello');          // ❌ raw string rejected
```

---

## 2. Opaque Types

```typescript
// Opaque type: the internal representation is HIDDEN from consumers
// They can only be created via a specific module's API

// src/types/opaque.ts
declare const OpaqueSymbol: unique symbol;
type Opaque<T, Token extends symbol = typeof OpaqueSymbol> = T & {
  readonly [OpaqueSymbol]: Token;
};

// Each opaque type uses a unique symbol for discrimination
declare const UserIdSymbol: unique symbol;
declare const PostIdSymbol: unique symbol;

type UserId2 = Opaque<string, typeof UserIdSymbol>;
type PostId2 = Opaque<string, typeof PostIdSymbol>;

// The module controls creation (consumers can't use `as` to bypass)
// src/user/user-id.ts
export function createUserId(raw: string): UserId2 {
  // validate format, then cast
  if (!raw.startsWith("user-")) throw new Error("Invalid user ID");
  return raw as unknown as UserId2;
}

export function getUserIdValue(id: UserId2): string {
  return id as unknown as string; // unwrap for serialization
}

// Why `as unknown as UserId2` instead of `as UserId2`:
// The unique symbol makes direct casting harder —
// you must go through `unknown` to acknowledge you're bypassing safety intentionally
```

---

## 3. The Builder Pattern with Type Accumulation

```typescript
// A builder that accumulates which fields have been set in the TYPE itself
// This makes calling .build() on an incomplete builder a COMPILE ERROR

type BuilderState = Record<string, unknown>;

class Builder<T extends BuilderState = {}> {
  private data: T;

  constructor(initial: T = {} as T) {
    this.data = initial;
  }

  // Each set() call returns a new Builder with an EXPANDED type
  set<K extends string, V>(key: K, value: V): Builder<T & Record<K, V>> {
    return new Builder({ ...this.data, [key]: value } as T & Record<K, V>);
  }

  // build() is only callable when T has the required fields
  build<Required extends T>(this: Builder<Required>): Required {
    return this.data as Required;
  }
}

// Usage — type tracks which fields have been set:
const builder = new Builder();
// builder: Builder<{}>

const withName = builder.set("name", "Alice");
// withName: Builder<{ name: string }>

const withAge = withName.set("age", 30);
// withAge: Builder<{ name: string; age: number }>

const result = withAge.build();
// result: { name: string; age: number }

// Practical: require specific fields before build() is callable
type RequiredUserFields = { name: string; email: string };

class UserBuilder<T extends Partial<RequiredUserFields> = {}> {
  private data: T = {} as T;

  withName(name: string): UserBuilder<T & { name: string }> {
    return Object.assign(new UserBuilder(), { data: { ...this.data, name } });
  }

  withEmail(email: string): UserBuilder<T & { email: string }> {
    return Object.assign(new UserBuilder(), { data: { ...this.data, email } });
  }

  withAge(age: number): UserBuilder<T & { age?: number }> {
    return Object.assign(new UserBuilder(), { data: { ...this.data, age } });
  }

  // build() only compiles if T includes both name AND email
  build(this: UserBuilder<T & RequiredUserFields>): T & RequiredUserFields {
    return this.data as T & RequiredUserFields;
  }
}

const user = new UserBuilder()
  .withName("Alice")
  .withEmail("alice@example.com")
  .build(); // ✅

const incomplete = new UserBuilder().withName("Bob").build(); // ❌ Property 'email' is missing in type '{ name: string }'
```

---

## 4. Type-Safe Error Handling (Result Type)

```typescript
// Result type: represent success OR failure without exceptions
type Ok<T> = { readonly ok: true; readonly value: T };
type Err<E> = { readonly ok: false; readonly error: E };
type Result<T, E = Error> = Ok<T> | Err<E>;

// Constructors
const ok = <T>(value: T): Ok<T> => ({ ok: true, value });
const err = <E>(error: E): Err<E> => ({ ok: false, error });

// Usage
async function parseConfig(raw: string): Promise<Result<Config, SyntaxError>> {
  try {
    return ok(JSON.parse(raw) as Config);
  } catch (e) {
    return err(e as SyntaxError);
  }
}

const result = await parseConfig(rawString);
if (result.ok) {
  result.value; // Config ✅
} else {
  result.error.message; // string ✅
}

// Chaining results (monadic style)
function mapResult<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => U,
): Result<U, E> {
  return result.ok ? ok(fn(result.value)) : result;
}

function flatMapResult<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => Result<U, E>,
): Result<U, E> {
  return result.ok ? fn(result.value) : result;
}

// Combining multiple results
function combineResults<T extends Result<unknown, unknown>[]>(
  ...results: T
): Result<
  { [K in keyof T]: T[K] extends Result<infer V, any> ? V : never },
  Error
> {
  const errors = results.filter((r) => !r.ok);
  if (errors.length > 0) {
    return err(new Error(`${errors.length} result(s) failed`));
  }
  return ok(results.map((r) => (r as Ok<unknown>).value) as any);
}
```

---

## 5. Phantom Types

```typescript
// Phantom types: type parameters that DON'T appear in the runtime value
// Used to track TYPE-LEVEL state without adding runtime data

// Example: form field validation states
type Unvalidated = "unvalidated";
type Validated = "validated";

// The _state type parameter is a phantom — never appears in the runtime value
type FormField<_State, T = string> = {
  value: T;
};

function createField<T = string>(value: T): FormField<Unvalidated, T> {
  return { value };
}

function validate<T>(
  field: FormField<Unvalidated, T>,
  predicate: (v: T) => boolean,
): FormField<Validated, T> | null {
  return predicate(field.value) ? { value: field.value } : null;
}

// Only validated fields can be submitted
function submitForm(email: FormField<Validated, string>): void {
  fetch("/api/submit", { body: JSON.stringify({ email: email.value }) });
}

const emailField = createField("alice@example.com");
// emailField: FormField<Unvalidated, string>

const validatedEmail = validate(emailField, (v) => v.includes("@"));
// validatedEmail: FormField<Validated, string> | null

if (validatedEmail) {
  submitForm(validatedEmail); // ✅
}
// submitForm(emailField); // ❌ Unvalidated not assignable to Validated

// Another example: currency phantom types
type Currency = "USD" | "EUR" | "GBP";
type Money<C extends Currency> = { amount: number; __currency: C };

function createMoney<C extends Currency>(
  amount: number,
  _currency: C,
): Money<C> {
  return { amount } as Money<C>;
}

function addMoney<C extends Currency>(a: Money<C>, b: Money<C>): Money<C> {
  return createMoney(a.amount + b.amount, undefined as any);
}

const usd = createMoney(100, "USD");
const eur = createMoney(50, "EUR");

addMoney(usd, usd); // ✅ same currency
// addMoney(usd, eur); // ❌ Money<'USD'> not assignable to Money<'EUR'>
```

---

## 6. Nominal Typing in a Structural System

```typescript
// TypeScript uses structural typing — two types with the same shape are compatible.
// Sometimes you need NOMINAL typing (identity matters, not just shape).

// Technique 1: unique symbol branding (most robust)
declare const _tag: unique symbol;
type Tagged<T, Tag extends string> = T & { readonly [_tag]: Tag };

// Technique 2: class-based nominal types
// Classes with private fields are NOMINALLY typed — two classes with the same
// shape but different private fields are NOT mutually assignable
class UserId {
  #id: string;
  constructor(id: string) {
    this.#id = id;
  }
  toString() {
    return this.#id;
  }
}
class PostId {
  #id: string;
  constructor(id: string) {
    this.#id = id;
  }
  toString() {
    return this.#id;
  }
}

function getUser(id: UserId): void {
  /* ... */
}
const userId = new UserId("u1");
const postId = new PostId("p1");

getUser(userId); // ✅
// getUser(postId); // ❌ PostId is not assignable to UserId
// (Even though they have the same shape, the private #id fields are unique to each class)

// Technique 3: Abstract class pattern
abstract class NominalId {
  abstract readonly __kind: string;
  constructor(readonly value: string) {}
}
class OrderId extends NominalId {
  readonly __kind = "OrderId" as const;
}
class CustomerId extends NominalId {
  readonly __kind = "CustomerId" as const;
}
```

---

## 7. Type-Safe Event Bus

```typescript
// Full type-safe event bus — covered in 12-mapped-and-conditional-types.md
// This version adds: typed subscriptions, priority, and one-time handlers

type EventMap = Record<string, unknown>;
type Handler<T> = (data: T) => void | Promise<void>;

class TypedEventBus<Events extends EventMap> {
  private subs = new Map<
    keyof Events,
    Set<{ handler: Handler<any>; once: boolean }>
  >();

  on<K extends keyof Events>(
    event: K,
    handler: Handler<Events[K]>,
  ): () => void {
    const set = this.subs.get(event) ?? new Set();
    const entry = { handler, once: false };
    set.add(entry);
    this.subs.set(event, set);
    return () => set.delete(entry); // unsubscribe function
  }

  once<K extends keyof Events>(event: K, handler: Handler<Events[K]>): void {
    const set = this.subs.get(event) ?? new Set();
    set.add({ handler, once: true });
    this.subs.set(event, set);
  }

  async emit<K extends keyof Events>(event: K, data: Events[K]): Promise<void> {
    const set = this.subs.get(event);
    if (!set) return;
    const toRemove: Set<{ handler: Handler<any>; once: boolean }> = new Set();
    for (const entry of set) {
      await entry.handler(data);
      if (entry.once) toRemove.add(entry);
    }
    toRemove.forEach((e) => set.delete(e));
  }
}
```

---

## 8. Exhaustive Pattern Matching

```typescript
// Pattern: ensure ALL cases of a discriminated union are handled

// Method 1: assertNever (throw at runtime for unhandled cases)
function assertNever(x: never, message?: string): never {
  throw new Error(message ?? `Unhandled value: ${JSON.stringify(x)}`);
}

type Shape = Circle | Square | Triangle;
function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default:
      return assertNever(shape); // ❌ error if Triangle is added but not handled
  }
}

// Method 2: exhaustive match helper (no runtime throw)
type Match<T extends { kind: string }, R> = {
  [K in T["kind"]]: (value: Extract<T, { kind: K }>) => R;
};

function match<T extends { kind: string }, R>(
  value: T,
  handlers: Match<T, R>,
): R {
  return (handlers as any)[value.kind](value);
}

// Usage:
const result = match(shape, {
  circle: (s) => Math.PI * s.radius ** 2,
  square: (s) => s.side ** 2,
  triangle: (s) => 0.5 * s.base * s.height,
  // ❌ TypeScript error if any case is missing
});
```

---

## 9. Type-Level Validation

```typescript
// Encode VALIDATION RULES in the type system so invalid values can't be created

// Validate at the type level using template literal and conditional types
type IsEmail<S extends string> = S extends `${string}@${string}.${string}`
  ? true
  : false;
type ValidEmail = IsEmail<"alice@example.com">; // true
type InvalidEmail = IsEmail<"not-an-email">; // false

// Type-safe range
type Range<Min extends number, Max extends number, T extends number = never> = {
  value: number;
  readonly __min: Min;
  readonly __max: Max;
};

function createRange<Min extends number, Max extends number>(
  value: number,
  min: Min,
  max: Max,
): Range<Min, Max> | null {
  if (value < min || value > max) return null;
  return { value } as Range<Min, Max>;
}

// Tuple length validation
type ExactLength<T extends unknown[], N extends number> = T["length"] extends N
  ? T
  : never;

function createTriple<T>(a: T, b: T, c: T): ExactLength<[T, T, T], 3> {
  return [a, b, c] as ExactLength<[T, T, T], 3>;
}

// Non-empty array type
type NonEmptyArray<T> = [T, ...T[]];

function first<T>(arr: NonEmptyArray<T>): T {
  return arr[0]; // Safe! TypeScript knows arr has at least one element
}
// first([]); // ❌ [] not assignable to NonEmptyArray

// Positive number brand
type Positive = Brand<number, "Positive">;
function createPositive(n: number): Positive {
  if (n <= 0) throw new Error(`Expected positive, got ${n}`);
  return n as Positive;
}
```

---

## 10. Recursive Type Utilities

```typescript
// Deep utilities that handle arbitrarily nested structures

// Paths to all leaf values in an object
type Paths<T, Prefix extends string = ""> = T extends object
  ? {
      [K in keyof T & string]: T[K] extends object
        ? Paths<T[K], `${Prefix}${K}.`>
        : `${Prefix}${K}`;
    }[keyof T & string]
  : Prefix;

type Config = { db: { host: string; port: number }; app: { debug: boolean } };
type ConfigPath = Paths<Config>; // 'db.host' | 'db.port' | 'app.debug'

// Get the type at a dot-separated path
type PathValue<T, P extends string> = P extends `${infer Key}.${infer Rest}`
  ? Key extends keyof T
    ? PathValue<T[Key], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;

type DbHost = PathValue<Config, "db.host">; // string
type AppDebug = PathValue<Config, "app.debug">; // boolean

// Type-safe get function
function getPath<T extends object, P extends Paths<T>>(
  obj: T,
  path: P,
): PathValue<T, P> {
  return path.split(".").reduce((acc: any, key) => acc[key], obj) as PathValue<
    T,
    P
  >;
}

getPath(config, "db.host"); // string ✅
getPath(config, "app.debug"); // boolean ✅
// getPath(config, 'db.name'); // ❌ not a valid path
```

---

## 11. Common Mistakes

### Mistake 1 — Branded types lost through operations

```typescript
// ❌ Arithmetic on branded numbers loses the brand
type Dollars = Brand<number, "Dollars">;
const price: Dollars = 9.99 as Dollars;
const doubled = price * 2; // `number`, NOT `Dollars`!

// ✅ Either re-brand the result or define safe arithmetic
function multiplyDollars(d: Dollars, factor: number): Dollars {
  return (d * factor) as Dollars;
}
```

### Mistake 2 — Using `as` to bypass brands defeats the purpose

```typescript
// ❌ Casting bypasses the entire point
const id = "post-123" as UserId; // TypeScript won't catch this

// ✅ Only create brands through constructor functions that validate
const id = createUserId("user-123"); // validates and brands
```

### Mistake 3 — Builder not returning `this` correctly in subclasses

```typescript
// ❌ Returning the base class type breaks method chaining in subclasses
class BaseBuilder {
  set(key: string, value: unknown): BaseBuilder {
    // returns BaseBuilder
    return this;
  }
}
class SpecificBuilder extends BaseBuilder {
  addExtra(): SpecificBuilder {
    return this;
  }
}
new SpecificBuilder().set("x", 1).addExtra(); // ❌ set returns BaseBuilder, no .addExtra()

// ✅ Use `this` as the return type for proper subclass chaining
class BaseBuilder {
  set(key: string, value: unknown): this {
    // returns `this` (polymorphic)
    return this;
  }
}
new SpecificBuilder().set("x", 1).addExtra(); // ✅
```

---

## 12. Exercises

### Exercise 1 — Implement a type-safe query builder

```typescript
// Build a SQL query builder where calling .build() is a type error
// unless both .from() and at least one .select() have been called.
// Use the type accumulation builder pattern.

// new QueryBuilder()
//   .from('users')           // marks 'hasFrom' in the type
//   .select('name', 'email') // marks 'hasSelect' in the type
//   .where('active = 1')
//   .build()                 // only compiles when hasFrom AND hasSelect are present
```

<details>
<summary>Solution</summary>

```typescript
type QueryState = {
  hasFrom: boolean;
  hasSelect: boolean;
};

class QueryBuilder<
  State extends QueryState = { hasFrom: false; hasSelect: false },
> {
  private sql: {
    table?: string;
    columns: string[];
    wheres: string[];
    limitVal?: number;
  } = { columns: [], wheres: [] };

  from(table: string): QueryBuilder<State & { hasFrom: true }> {
    this.sql.table = table;
    return this as any;
  }

  select(...columns: string[]): QueryBuilder<State & { hasSelect: true }> {
    this.sql.columns.push(...columns);
    return this as any;
  }

  where(condition: string): this {
    this.sql.wheres.push(condition);
    return this;
  }

  limit(n: number): this {
    this.sql.limitVal = n;
    return this;
  }

  build(this: QueryBuilder<{ hasFrom: true; hasSelect: true }>): string {
    const { table, columns, wheres, limitVal } = this.sql;
    let query = `SELECT ${columns.join(", ")} FROM ${table}`;
    if (wheres.length) query += ` WHERE ${wheres.join(" AND ")}`;
    if (limitVal) query += ` LIMIT ${limitVal}`;
    return query;
  }
}

// ✅
const sql = new QueryBuilder()
  .from("users")
  .select("name", "email")
  .where("active = 1")
  .limit(10)
  .build(); // "SELECT name, email FROM users WHERE active = 1 LIMIT 10"

// ❌
new QueryBuilder().from("users").build(); // Error: hasSelect is false
new QueryBuilder().select("*").build(); // Error: hasFrom is false
```

</details>

---

## 🔗 Related Topics

- [`11-advanced-generics.md`](./11-advanced-generics.md) — `infer` and conditional types
- [`12-mapped-and-conditional-types.md`](./12-mapped-and-conditional-types.md) — Building type utilities
- [`javascript-core/27-proxy-reflect-and-metaprogramming.md`](../javascript-core/27-proxy-reflect-and-metaprogramming.md) — Runtime metaprogramming complement
