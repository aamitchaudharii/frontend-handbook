# 03 — Interfaces and Type Aliases

> **"Interfaces and type aliases are both ways to name a shape. They overlap so much that the real question isn't 'which is better?' — it's 'which fits THIS situation?' Interfaces are for object shapes that get extended and implemented. Type aliases are for everything else: unions, primitives, computed types, and shapes that won't grow a hierarchy."**

🟢 **Level: Beginner**

---

## 📚 Table of Contents

1. [Defining Interfaces](#1-defining-interfaces)
2. [Defining Type Aliases](#2-defining-type-aliases)
3. [interface vs type — The Differences](#3-interface-vs-type--the-differences)
4. [Extending Interfaces](#4-extending-interfaces)
5. [Intersection Types (type extension)](#5-intersection-types-type-extension)
6. [Declaration Merging (interfaces only)](#6-declaration-merging-interfaces-only)
7. [Implementing Interfaces in Classes](#7-implementing-interfaces-in-classes)
8. [When to Use Each](#8-when-to-use-each)
9. [Common Mistakes](#9-common-mistakes)
10. [Exercises](#10-exercises)

---

## 1. Defining Interfaces

```typescript
// interface: defines the shape of an OBJECT
interface User {
  id: number;
  name: string;
  email: string;
  age?: number; // optional
  readonly createdAt: Date; // read-only
}

// Using the interface
const alice: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  createdAt: new Date(),
};

// Methods in interfaces
interface Repository<T> {
  findById(id: number): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: number): Promise<void>;
}

// Function signatures in interfaces
interface Comparator<T> {
  (a: T, b: T): number; // call signature
}

interface StringComparator extends Comparator<string> {}
```

---

## 2. Defining Type Aliases

```typescript
// type: names ANY type expression (not just objects)
type ID = string | number;
type Nullable<T> = T | null;
type Callback = () => void;

// Object shape (same as interface for simple cases)
type User = {
  id: number;
  name: string;
  email: string;
};

// Types can express things interfaces CANNOT:
type StringOrNumber = string | number; // union
type Both = TypeA & TypeB; // intersection
type Coords = [number, number]; // tuple
type Matrix = number[][]; // array type
type EventName = `on${string}`; // template literal type
type Keys = keyof User; // keyof
type UserName = User["name"]; // indexed access type
```

---

## 3. interface vs type — The Differences

```typescript
// DIFFERENCE 1: Declaration merging (only interface)
// Declaring the same interface twice MERGES them
interface Window {
  myCustomProp: string; // adds to the existing Window interface
}
// type Window = { myCustomProp: string }; // ❌ Error: duplicate identifier

// DIFFERENCE 2: Extending syntax
interface Animal {
  name: string;
}
interface Dog extends Animal {
  breed: string;
} // interface: extends keyword

type AnimalType = { name: string };
type DogType = AnimalType & { breed: string }; // type: & intersection

// DIFFERENCE 3: Union and other complex types (only type)
type Shape = Circle | Square; // ✅ union
// interface Shape = Circle | Square; // ❌ can't do unions with interface

// DIFFERENCE 4: Computed/mapped types (only type)
type Readonly<T> = { readonly [K in keyof T]: T[K] }; // ✅
// interface can't use mapped type syntax

// DIFFERENCE 5: Implementing in classes (both work)
interface Serializable {
  serialize(): string;
}
class User implements Serializable {
  serialize() {
    return JSON.stringify(this);
  }
}
// type Serializable = { serialize(): string; }; // can also be implemented

// SIMILARITY: Structural typing — both are just shapes
// TypeScript uses structural (duck) typing: if it has the right shape, it fits
interface HasName {
  name: string;
}
type HasAge = { age: number };

function greet(user: HasName) {
  return user.name;
}
greet({ name: "Alice", age: 30 }); // ✅ extra properties OK when passed inline
// BUT: excess property checking kicks in for object literals:
// const u: HasName = { name: 'Alice', age: 30 }; // ❌ excess property 'age'
const u = { name: "Alice", age: 30 };
greet(u); // ✅ no excess property check when passing via a variable
```

---

## 4. Extending Interfaces

```typescript
interface Animal {
  name: string;
  sound(): string;
}

interface Pet extends Animal {
  owner: string;
}

interface Dog extends Pet {
  breed: string;
}

// A Dog must satisfy all three interfaces
const rex: Dog = {
  name: "Rex",
  owner: "Alice",
  breed: "Labrador",
  sound() {
    return "Woof";
  },
};

// Multiple inheritance (interfaces only)
interface Flyable {
  fly(): void;
}
interface Swimmable {
  swim(): void;
}

interface Duck extends Animal, Flyable, Swimmable {}

// Extending with modifications (override a property's type narrower)
interface BasicUser {
  role: string;
}
interface AdminUser extends BasicUser {
  role: "admin"; // narrowed from string to literal 'admin'
  permissions: string[];
}
```

---

## 5. Intersection Types (type extension)

```typescript
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type SoftDeletable = {
  deletedAt: Date | null;
};

// Intersection: must satisfy ALL types
type AuditedEntity = Timestamped & SoftDeletable;

type User = {
  id: number;
  name: string;
} & Timestamped;

const user: User = {
  id: 1,
  name: "Alice",
  createdAt: new Date(),
  updatedAt: new Date(),
};

// Useful pattern: add common fields to any type
type WithId<T> = T & { id: number };
type WithTimestamps<T> = T & { createdAt: Date; updatedAt: Date };
type WithPagination<T> = T & { page: number; pageSize: number; total: number };

type UserCreateInput = WithTimestamps<{ name: string; email: string }>;
```

---

## 6. Declaration Merging (interfaces only)

```typescript
// Declaration merging is the PRIMARY reason to prefer interface over type
// for object shapes that may need to be extended externally (e.g., in libraries)

// Use case 1: extending third-party types
// Extend Express's Request to include our authenticated user
declare module "express" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
// Now in route handlers: req.user is typed correctly ✅

// Use case 2: extending global types (e.g., Window)
interface Window {
  analytics: AnalyticsInstance;
  __DEV__: boolean;
}

// Use case 3: augmenting library interfaces
import "some-library";
declare module "some-library" {
  interface PluginOptions {
    myCustomOption?: string;
  }
}

// ⚠️ Be careful: merging interface properties must be compatible
interface A {
  x: string;
}
interface A {
  x: number;
} // ❌ Error: property 'x' must be the same type
interface A {
  y: string;
} // ✅ adding a NEW property is fine
```

---

## 7. Implementing Interfaces in Classes

```typescript
interface Shape {
  area(): number;
  perimeter(): number;
  toString(): string;
}

class Circle implements Shape {
  constructor(private radius: number) {}

  area() {
    return Math.PI * this.radius ** 2;
  }
  perimeter() {
    return 2 * Math.PI * this.radius;
  }
  toString() {
    return `Circle(r=${this.radius})`;
  }
}

class Rectangle implements Shape {
  constructor(
    private width: number,
    private height: number,
  ) {}

  area() {
    return this.width * this.height;
  }
  perimeter() {
    return 2 * (this.width + this.height);
  }
  toString() {
    return `Rect(${this.width}×${this.height})`;
  }
}

// Interface as a contract — any implementing class satisfies it
function printShape(shape: Shape): void {
  console.log(`${shape}: area=${shape.area().toFixed(2)}`);
}

printShape(new Circle(5));
printShape(new Rectangle(3, 4));

// A class can implement MULTIPLE interfaces
interface Serializable {
  serialize(): string;
}
interface Cloneable {
  clone(): this;
}

class Config implements Serializable, Cloneable {
  constructor(public data: Record<string, unknown>) {}
  serialize() {
    return JSON.stringify(this.data);
  }
  clone() {
    return new Config({ ...this.data }) as this;
  }
}
```

---

## 8. When to Use Each

```
USE interface WHEN:
  ✅ Defining the shape of an object that may be extended (extends keyword)
  ✅ Creating a contract for a class to implement (implements keyword)
  ✅ The type may need declaration merging (library types, module augmentation)
  ✅ You want error messages to say "interface MyType" (slightly clearer in some cases)

USE type WHEN:
  ✅ Creating union types: type Result = Success | Failure
  ✅ Creating intersection types without inheritance hierarchy
  ✅ Naming tuple types: type Point = [number, number]
  ✅ Using mapped types, conditional types, template literal types
  ✅ Naming primitive types or function types: type ID = string
  ✅ You want to avoid accidental declaration merging

RULE OF THUMB (team consensus):
  Many teams just pick one and use it consistently.
  The most common convention in React codebases:
  - type for component Props: type ButtonProps = { ... }
  - interface for domain models: interface User { ... }
```

---

## 9. Common Mistakes

### Mistake 1 — Excess property checking surprises

```typescript
interface User {
  name: string;
  age: number;
}

// ❌ Excess property check catches this:
const u: User = { name: "Alice", age: 30, email: "alice@ex.com" };
// Error: Object literal may only specify known properties

// ✅ But this is allowed (structural check, not excess property check):
const obj = { name: "Alice", age: 30, email: "alice@ex.com" };
const u: User = obj; // OK — obj is a superset of User's shape

// This is intentional TS behavior: strict on literal objects, lenient on variables
```

### Mistake 2 — Conflating interface implementation with type checking

```typescript
// TS uses STRUCTURAL typing — you don't need to explicitly implement an interface
interface Greetable {
  greet(): string;
}

// This satisfies Greetable WITHOUT `implements`:
const obj = {
  greet() {
    return "Hello!";
  },
};

function sayHello(g: Greetable) {
  console.log(g.greet());
}
sayHello(obj); // ✅ works — it has the right shape

// `implements` is useful for explicit contract + class tooling,
// but it's not required for structural compatibility
```

---

## 10. Exercises

### Exercise 1 — Design a type hierarchy

```typescript
// Model a simple CMS with these constraints:
// - All content items have: id (number), title (string), createdAt (Date)
// - Articles also have: body (string), tags (string[])
// - Videos also have: url (string), duration (number in seconds)
// - Both should extend a common ContentItem base
// Use interface and extends.
```

<details>
<summary>Solution</summary>

```typescript
interface ContentItem {
  id: number;
  title: string;
  createdAt: Date;
}

interface Article extends ContentItem {
  body: string;
  tags: string[];
}

interface Video extends ContentItem {
  url: string;
  duration: number;
}

// Usage
const article: Article = {
  id: 1,
  title: "Hello World",
  createdAt: new Date(),
  body: "Content here",
  tags: ["typescript", "tutorial"],
};
const video: Video = {
  id: 2,
  title: "TS Course",
  createdAt: new Date(),
  url: "https://example.com/video.mp4",
  duration: 3600,
};

function render(item: ContentItem) {
  console.log(item.title);
}
render(article); // ✅
render(video); // ✅
```

</details>

---

## 🔗 Related Topics

- [`04-functions-in-typescript.md`](./04-functions-in-typescript.md) — Typing functions and callbacks
- [`05-union-and-intersection-types.md`](./05-union-and-intersection-types.md) — Combining types with | and &
- [`07-generics.md`](./07-generics.md) — Generic interfaces and type aliases
