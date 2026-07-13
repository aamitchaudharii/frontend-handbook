# 13 — Template Literal Types

> **"Template literal types bring string manipulation into the type system. They're how TypeScript knows that an event named `'click'` maps to a handler called `onClick`, that a CSS property `margin-top` maps to `marginTop`, or that a route string `/users/:id` has an `:id` parameter. This is string pattern matching at compile time."**

🟠 **Level: Advanced**

---

## 📚 Table of Contents

1. [Basic Template Literal Types](#1-basic-template-literal-types)
2. [Combining with Union Types](#2-combining-with-union-types)
3. [Intrinsic String Utility Types](#3-intrinsic-string-utility-types)
4. [infer with Template Literals](#4-infer-with-template-literals)
5. [Key Remapping with Template Literals](#5-key-remapping-with-template-literals)
6. [Parsing String Patterns](#6-parsing-string-patterns)
7. [Real-World Patterns](#7-real-world-patterns)
8. [Common Mistakes](#8-common-mistakes)
9. [Exercises](#9-exercises)

---

## 1. Basic Template Literal Types

```typescript
// Template literal types use the same backtick syntax as template literal strings,
// but at the TYPE level

type Greeting = `Hello, ${string}!`; // matches any "Hello, X!" string

const a: Greeting = "Hello, World!"; // ✅
const b: Greeting = "Hello, Alice!"; // ✅
const c: Greeting = "Goodbye, World!"; // ❌ doesn't match the pattern

// With specific literals:
type Name = "Alice" | "Bob";
type Hello = `Hello, ${Name}!`; // 'Hello, Alice!' | 'Hello, Bob!'

// With primitive types:
type EmailLike = `${string}@${string}.${string}`;
type CSSLength = `${number}px` | `${number}em` | `${number}rem` | `${number}%`;
type HexColor = `#${string}`;
type DataAttr = `data-${string}`;

// They DON'T validate values at a deep level — just pattern matching:
const weird: CSSLength = `${NaN}px`; // ✅ TypeScript sees "numberLike"px
const valid: CSSLength = "16px"; // ✅
```

---

## 2. Combining with Union Types

```typescript
// When a template literal contains a union, it DISTRIBUTES over all combinations

type Color = "red" | "green" | "blue";
type Size = "sm" | "md" | "lg";

type ColorSize = `${Color}-${Size}`;
// 'red-sm' | 'red-md' | 'red-lg' | 'green-sm' | 'green-md' | 'green-lg' | 'blue-sm' | 'blue-md' | 'blue-lg'
// (3 × 3 = 9 combinations)

// CSS border sides
type Side = "top" | "right" | "bottom" | "left";
type CSSBorder = `border-${Side}`;
// 'border-top' | 'border-right' | 'border-bottom' | 'border-left'

// Event types from a list of HTML elements and event names
type Element = "button" | "input" | "form";
type EventType = "click" | "focus" | "submit";
type ElementEvent = `${Element}:${EventType}`;
// 'button:click' | 'button:focus' | ... (9 combinations)

// Practical: generate Redux action type names
type Domain = "users" | "products" | "orders";
type Op = "fetch" | "create" | "update" | "delete";
type ActionType = `${Uppercase<Domain>}/${Uppercase<Op>}`;
// 'USERS/FETCH' | 'USERS/CREATE' | ... (12 combinations)
```

---

## 3. Intrinsic String Utility Types

```typescript
// TypeScript provides 4 built-in string transformation types:

type U = Uppercase<"hello">; // 'HELLO'
type L = Lowercase<"HELLO">; // 'hello'
type C = Capitalize<"hello">; // 'Hello'
type N = Uncapitalize<"Hello">; // 'hello'

// These distribute over unions:
type Keys = "name" | "age" | "email";
type Upper = Uppercase<Keys>; // 'NAME' | 'AGE' | 'EMAIL'
type Cap = Capitalize<Keys>; // 'Name' | 'Age' | 'Email'

// Combining with template literals:
type EventName = "click" | "focus" | "blur" | "change" | "submit";
type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur' | 'onChange' | 'onSubmit'

// DOM handler types from events
type DOMHandlers = {
  [K in EventName as `on${Capitalize<K>}`]: (event: Event) => void;
};
// { onClick: ...; onFocus: ...; onBlur: ...; onChange: ...; onSubmit: ... }

// SCREAMING_SNAKE_CASE from camelCase (partial — full version needs infer)
type ToConst<S extends string> = Uppercase<S>;
// camelCase → CAMELCASE (rough approximation)
```

---

## 4. infer with Template Literals

```typescript
// Extract parts of a string type using infer inside template literal patterns

// Extract the domain from an email pattern
type ExtractDomain<T extends string> = T extends `${string}@${infer Domain}`
  ? Domain
  : never;
type D1 = ExtractDomain<"alice@example.com">; // 'example.com'
type D2 = ExtractDomain<"bob@test.org">; // 'test.org'
type D3 = ExtractDomain<"notanemail">; // never

// Extract URL parameter names
type ExtractParam<S extends string> =
  S extends `${string}:${infer Param}/${string}`
    ? Param
    : S extends `${string}:${infer Param}`
      ? Param
      : never;
type P1 = ExtractParam<"/users/:id">; // 'id'
type P2 = ExtractParam<"/posts/:slug/comments">; // 'slug'

// Extract all params from a route (recursive)
type ExtractParams<S extends string> =
  S extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParams<`/${Rest}`>
    : S extends `${string}:${infer Param}`
      ? Param
      : never;

type Params = ExtractParams<"/users/:userId/posts/:postId">;
// 'userId' | 'postId'

// Build a params object type from a route string
type RouteParams<S extends string> = {
  [K in ExtractParams<S>]: string;
};
type UserPostParams = RouteParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }

// Convert snake_case to camelCase
type SnakeToCamel<S extends string> = S extends `${infer Head}_${infer Tail}`
  ? `${Head}${Capitalize<SnakeToCamel<Tail>>}`
  : S;

type C1 = SnakeToCamel<"hello_world">; // 'helloWorld'
type C2 = SnakeToCamel<"first_name_last_name">; // 'firstNameLastName'

// Convert camelCase to snake_case (harder — requires checking each char)
type CamelToSnake<S extends string> = S extends `${infer Head}${infer Tail}`
  ? Head extends Uppercase<Head>
    ? `_${Lowercase<Head>}${CamelToSnake<Tail>}`
    : `${Head}${CamelToSnake<Tail>}`
  : S;
type S1 = CamelToSnake<"helloWorld">; // 'hello_world'
type S2 = CamelToSnake<"firstName">; // 'first_name'
```

---

## 5. Key Remapping with Template Literals

```typescript
// Use template literals in `as` clause to transform keys

// Add prefix to all keys
type Prefixed<T, P extends string> = {
  [K in keyof T & string as `${P}${Capitalize<K>}`]: T[K];
};
type PUser = Prefixed<{ name: string; age: number }, "user">;
// { userName: string; userAge: number }

// Generate getter/setter pairs
type WithGettersAndSetters<T> = {
  [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K];
} & { [K in keyof T & string as `set${Capitalize<K>}`]: (v: T[K]) => void };

type UserAPI = WithGettersAndSetters<{ name: string; age: number }>;
// { getName: () => string; setName: (v: string) => void;
//   getAge: () => number; setAge: (v: number) => void }

// Transform snake_case object keys to camelCase
type CamelizeKeys<T> = {
  [K in keyof T & string as SnakeToCamel<K>]: T[K];
};
type DbUser = { first_name: string; last_name: string; email_address: string };
type ApiUser = CamelizeKeys<DbUser>;
// { firstName: string; lastName: string; emailAddress: string }

// Route to handler mapping
type Routes = "/users" | "/users/:id" | "/posts" | "/posts/:id";
type RouteHandler = {
  [K in Routes as `handle${Capitalize<K extends `/${infer P}` ? P : K>}`]: () => void;
};
```

---

## 6. Parsing String Patterns

```typescript
// Use recursive template literal types to parse complex string patterns

// Split a string by delimiter into a tuple
type Split<
  S extends string,
  D extends string,
> = S extends `${infer Head}${D}${infer Tail}`
  ? [Head, ...Split<Tail, D>]
  : [S];

type Words = Split<"hello world foo", " ">; // ['hello', 'world', 'foo']
type Parts = Split<"a.b.c.d", ".">; // ['a', 'b', 'c', 'd']

// Join a tuple of strings
type Join<T extends string[], D extends string = ""> = T extends [
  infer H extends string,
  ...infer Tail extends string[],
]
  ? Tail extends []
    ? H
    : `${H}${D}${Join<Tail, D>}`
  : "";

type J1 = Join<["hello", "world"], " ">; // 'hello world'
type J2 = Join<["a", "b", "c"], ".">; // 'a.b.c'

// Replace a substring
type Replace<
  S extends string,
  From extends string,
  To extends string,
> = S extends `${infer Before}${From}${infer After}`
  ? `${Before}${To}${After}`
  : S;
type R1 = Replace<"hello world", "world", "TypeScript">; // 'hello TypeScript'

// Trim whitespace
type TrimLeft<S extends string> = S extends ` ${infer T}` ? TrimLeft<T> : S;
type TrimRight<S extends string> = S extends `${infer T} ` ? TrimRight<T> : S;
type Trim<S extends string> = TrimLeft<TrimRight<S>>;
type T1 = Trim<"  hello  ">; // 'hello'
```

---

## 7. Real-World Patterns

```typescript
// PATTERN 1: Type-safe i18n keys
type Messages = {
  "greeting.hello": string;
  "greeting.goodbye": string;
  "error.notFound": string;
  "error.forbidden": string;
};

type MessageKey = keyof Messages; // 'greeting.hello' | ...

// Extract only 'greeting.*' keys
type GreetingKey = Extract<MessageKey, `greeting.${string}`>;
// 'greeting.hello' | 'greeting.goodbye'

function t(key: MessageKey): string {
  return messages[key];
}
t("greeting.hello"); // ✅
t("greeting.wrong"); // ❌

// PATTERN 2: CSS-in-JS property names
type CSSProperty = keyof CSSStyleDeclaration & string;
type CSSValue<P extends CSSProperty> = CSSStyleDeclaration[P];

// PATTERN 3: tRPC-style typed API routes
type ApiRoutes = {
  "user.get": { input: { id: string }; output: User };
  "user.list": { input: { page: number }; output: User[] };
  "user.create": { input: Omit<User, "id">; output: User };
};

type ApiClient = {
  [Route in keyof ApiRoutes]: (
    input: ApiRoutes[Route]["input"],
  ) => Promise<ApiRoutes[Route]["output"]>;
};

// PATTERN 4: Env variable type safety
type EnvVar = "DATABASE_URL" | "API_KEY" | "PORT" | "NODE_ENV";
type Env = Record<EnvVar, string>;

function getEnv<K extends EnvVar>(key: K): string {
  const val = process.env[key];
  if (!val) throw new Error(`Missing env: ${key}`);
  return val;
}
getEnv("DATABASE_URL"); // ✅
// getEnv('UNKNOWN_VAR'); // ❌
```

---

## 8. Common Mistakes

### Mistake 1 — Template literals don't validate string values at runtime

```typescript
// ❌ This is a type-level pattern only
type EmailLike = `${string}@${string}`;
const email: EmailLike = "not-an-email"; // TypeScript errors ✅
// But at runtime, TypeScript has been erased — you still need actual validation

// ✅ Use both the type AND a runtime check
function validateEmail(s: string): s is EmailLike {
  return /^[^\s@]+@[^\s@]+$/.test(s);
}
```

### Mistake 2 — Exponential combinations blow up the type

```typescript
// ❌ This creates 4^4 = 256 combinations — TypeScript may warn or slow down
type Axis = "x" | "y" | "z" | "w";
type Coords = `${Axis}${Axis}${Axis}${Axis}`; // 256 members — TS warns at 100,000

// ✅ Keep union sizes reasonable; use string for very open-ended patterns
type Coords2 = `${Axis}${Axis}`; // 16 members — fine
```

---

## 9. Exercises

### Exercise 1 — Type-safe event emitter with template literals

```typescript
// Given these event names: 'user' | 'product' | 'order'
// And operations: 'created' | 'updated' | 'deleted'
// Build:
// 1. EventName type: 'user:created' | 'user:updated' | ... (9 total)
// 2. EventHandlers type: { 'on:user:created': Handler, ... }
// 3. A typed emit function that only accepts valid EventName values
```

<details>
<summary>Solution</summary>

```typescript
type Domain = "user" | "product" | "order";
type Operation = "created" | "updated" | "deleted";
type EventName = `${Domain}:${Operation}`;
// 'user:created' | 'user:updated' | ... (9 combinations)

type Handler = (timestamp: number) => void;
type EventHandlers = {
  [K in EventName as `on:${K}`]: Handler;
};

class TypedEmitter {
  private handlers: Partial<EventHandlers> = {};

  on<K extends EventName>(event: K, handler: Handler): void {
    (this.handlers as any)[`on:${event}`] = handler;
  }

  emit(event: EventName): void {
    const handler = (this.handlers as any)[`on:${event}`];
    handler?.(Date.now());
  }
}

const emitter = new TypedEmitter();
emitter.on("user:created", (ts) => console.log("User created at", ts)); // ✅
emitter.emit("user:created"); // ✅
// emitter.emit('user:purchased'); // ❌
```

</details>

---

## 🔗 Related Topics

- [`12-mapped-and-conditional-types.md`](./12-mapped-and-conditional-types.md) — Key remapping
- [`11-advanced-generics.md`](./11-advanced-generics.md) — `infer` foundations
- [`10-enums-and-literal-types.md`](./10-enums-and-literal-types.md) — Literal types
