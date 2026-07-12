# 09 — Classes and Access Modifiers

> **"TypeScript classes are JavaScript classes with a type layer on top. Access modifiers, abstract classes, and parameter properties aren't just syntax sugar — they encode design intent directly in the code. `private` says 'this is an implementation detail.' `abstract` says 'you must provide this.' `readonly` says 'set it once and never again.'"**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [Class Basics in TypeScript](#1-class-basics-in-typescript)
2. [Access Modifiers](#2-access-modifiers)
3. [Parameter Properties](#3-parameter-properties)
4. [readonly Properties](#4-readonly-properties)
5. [Static Members](#5-static-members)
6. [Abstract Classes](#6-abstract-classes)
7. [implements vs extends](#7-implements-vs-extends)
8. [Private Fields — # vs private](#8-private-fields----vs-private)
9. [Class Types and Structural Typing](#9-class-types-and-structural-typing)
10. [Common Patterns](#10-common-patterns)
11. [Common Mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)

---

## 1. Class Basics in TypeScript

```typescript
class Animal {
  name: string; // property declaration (required in TS with strict mode)
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  speak(): string {
    return `${this.name} makes a sound`;
  }

  toString(): string {
    return `${this.name} (${this.age})`;
  }
}

const cat = new Animal("Whiskers", 3);
cat.speak(); // 'Whiskers makes a sound'
cat.name; // 'Whiskers'

// TypeScript enforces property initialization
class Bad {
  name: string; // ❌ Property 'name' has no initializer and is not definitely
  //    assigned in the constructor (with strictPropertyInitialization)
}

// ✅ Options:
class Good {
  name: string = ""; // initialize with default
  name!: string; // ! tells TS "I'll handle this" (use carefully)
  name: string | undefined; // widen type to allow undefined
}
```

---

## 2. Access Modifiers

```typescript
class BankAccount {
  public owner: string; // accessible everywhere (default)
  protected balance: number; // accessible in this class and subclasses
  private pin: string; // accessible ONLY in this class

  constructor(owner: string, initialBalance: number, pin: string) {
    this.owner = owner;
    this.balance = initialBalance;
    this.pin = pin;
  }

  // Public method (accessible everywhere)
  getBalance(): number {
    return this.balance;
  }

  // Private method (only callable inside this class)
  private validatePin(input: string): boolean {
    return input === this.pin;
  }

  // Protected method (accessible in subclasses)
  protected applyFee(amount: number): void {
    this.balance -= amount;
  }

  withdraw(amount: number, pin: string): boolean {
    if (!this.validatePin(pin)) return false;
    if (amount > this.balance) return false;
    this.balance -= amount;
    this.applyFee(2.5); // transaction fee
    return true;
  }
}

class SavingsAccount extends BankAccount {
  withdraw(amount: number, pin: string): boolean {
    // ✅ Can access protected members from parent
    this.applyFee(0); // override: no fee for savings
    return super.withdraw(amount, pin);
  }

  getInterest(): number {
    return this.balance * 0.05; // ✅ can access protected balance
  }

  // ❌ Cannot access private members from parent
  // getPin() { return this.pin; } // Error: private
}

const account = new BankAccount("Alice", 1000, "1234");
account.owner; // ✅ public
// account.balance;   // ❌ protected
// account.pin;       // ❌ private
```

---

## 3. Parameter Properties

```typescript
// ❌ Verbose: declare, then assign in constructor
class User {
  name: string;
  email: string;
  role: string;

  constructor(name: string, email: string, role: string) {
    this.name = name;
    this.email = email;
    this.role = role;
  }
}

// ✅ Parameter properties: declare AND assign in the constructor signature
class User {
  constructor(
    public name: string,
    public email: string,
    protected role: string = "user",
    private id: number = Math.random(),
  ) {}
  // All four are automatically declared as class properties
}

const user = new User("Alice", "alice@example.com");
user.name; // 'Alice'
user.email; // 'alice@example.com'
// user.id;  // ❌ private

// Mix parameter properties with regular properties
class Config {
  readonly createdAt = new Date(); // regular property with initializer

  constructor(
    public host: string,
    public port: number = 3000,
    private debug: boolean = false,
  ) {}
}
```

---

## 4. readonly Properties

```typescript
class Point {
  readonly x: number;
  readonly y: number;

  constructor(x: number, y: number) {
    this.x = x; // ✅ can assign in constructor
    this.y = y;
  }

  translate(dx: number, dy: number): Point {
    // this.x += dx; // ❌ cannot reassign readonly
    return new Point(this.x + dx, this.y + dy); // return a new Point instead
  }
}

const p = new Point(1, 2);
// p.x = 5; // ❌ Cannot assign to 'x' because it is a read-only property

// readonly vs const:
// const: the BINDING (variable) is immutable
// readonly: the PROPERTY is immutable (still a new allocation per instance)

// Readonly class with parameter properties (very common pattern)
class ImmutableConfig {
  constructor(
    public readonly host: string,
    public readonly port: number,
    public readonly timeout: number = 5000,
  ) {}
}
```

---

## 5. Static Members

```typescript
class MathHelper {
  static readonly PI = 3.14159265358979;

  static circleArea(radius: number): number {
    return MathHelper.PI * radius ** 2;
  }

  static factorial(n: number): number {
    return n <= 1 ? 1 : n * MathHelper.factorial(n - 1);
  }
}

MathHelper.circleArea(5); // ✅ called on the class, not an instance
// new MathHelper().circleArea(5); // works but is misleading

// Singleton pattern using static
class Database {
  private static instance: Database | null = null;
  private constructor(private url: string) {}

  static getInstance(url?: string): Database {
    if (!Database.instance) {
      if (!url) throw new Error("URL required for first initialization");
      Database.instance = new Database(url);
    }
    return Database.instance;
  }

  query(sql: string): unknown[] {
    return [];
  }
}

const db1 = Database.getInstance("postgres://localhost/mydb");
const db2 = Database.getInstance(); // returns same instance
db1 === db2; // true

// Static with generics
class Registry<T> {
  private static entries = new Map<string, unknown>();

  static register<T>(key: string, value: T): void {
    Registry.entries.set(key, value);
  }

  static get<T>(key: string): T | undefined {
    return Registry.entries.get(key) as T | undefined;
  }
}
```

---

## 6. Abstract Classes

```typescript
// Abstract class: cannot be instantiated, but defines a contract for subclasses
abstract class Shape {
  // Concrete method (shared implementation)
  toString(): string {
    return `${this.constructor.name} with area ${this.area().toFixed(2)}`;
  }

  // Abstract methods: MUST be implemented by subclasses
  abstract area(): number;
  abstract perimeter(): number;

  // Abstract property
  abstract readonly name: string;
}

class Circle extends Shape {
  readonly name = "Circle";

  constructor(private radius: number) {
    super();
  }

  area() {
    return Math.PI * this.radius ** 2;
  }
  perimeter() {
    return 2 * Math.PI * this.radius;
  }
}

class Rectangle extends Shape {
  readonly name = "Rectangle";

  constructor(
    private w: number,
    private h: number,
  ) {
    super();
  }

  area() {
    return this.w * this.h;
  }
  perimeter() {
    return 2 * (this.w + this.h);
  }
}

// new Shape(); // ❌ Cannot create an instance of an abstract class

const shapes: Shape[] = [new Circle(5), new Rectangle(3, 4)];
shapes.forEach((s) => console.log(s.toString())); // uses concrete implementations

// Abstract with generics
abstract class Repository<T, ID = string> {
  abstract findById(id: ID): Promise<T | null>;
  abstract save(entity: T): Promise<T>;
  abstract delete(id: ID): Promise<void>;

  async findOrFail(id: ID): Promise<T> {
    const entity = await this.findById(id);
    if (!entity) throw new Error(`Entity ${id} not found`);
    return entity;
  }
}
```

---

## 7. implements vs extends

```typescript
interface Printable {
  print(): void;
}
interface Serializable {
  serialize(): string;
}

class Animal {
  constructor(public name: string) {}
  speak() {
    return `${this.name} speaks`;
  }
}

// extends: inherits implementation AND type
// implements: only checks the type contract (no implementation inherited)
class Dog extends Animal implements Printable, Serializable {
  print() {
    console.log(this.name);
  }
  serialize() {
    return JSON.stringify({ name: this.name });
  }
}

// A class can extends ONE class but implements MANY interfaces
// class Multi extends A, B {} // ❌ — single inheritance only
class Multi extends Animal implements Printable, Serializable {
  print() {
    console.log(this.name);
  }
  serialize() {
    return this.name;
  }
}

// interface can extend a class (unusual but valid)
interface DogInterface extends Animal {
  breed: string;
}
// DogInterface includes all Animal members + breed
```

---

## 8. Private Fields — # vs private

```typescript
// TypeScript's `private` keyword: compile-time only
class TsPrivate {
  private secret = 'hidden';
}
const obj = new TsPrivate();
// obj.secret; // ❌ TypeScript error
(obj as any).secret; // ✅ accessible at runtime — `private` is only compile-time

// JavaScript's `#` private fields: RUNTIME enforcement
class JsPrivate {
  #secret = 'truly hidden';

  getSecret() { return this.#secret; }
}
const obj2 = new JsPrivate();
// obj2.#secret;        // ❌ Syntax error (even at runtime)
// (obj2 as any).secret; // undefined — #fields don't appear on the object
obj2.getSecret();      // 'truly hidden'

// KEY DIFFERENCE:
// private → compile-time check only, erased in JS, accessible via (x as any)
// #       → runtime enforcement, truly inaccessible, appears in JS output

// When to use which:
// # for truly sensitive data or when you need guaranteed encapsulation
// private for most cases (compile-time is sufficient, better tooling support)

// Both work as parameter properties:
class Config {
  constructor(
    private   tsPrivate: string,  // `private` keyword
    readonly  #jsPrivate: string  // can't combine # with parameter property shorthand
  ) {}                            // ← actually you CAN'T use # in parameter properties
}                                  // Must declare separately:

class Config2 {
  readonly #key: string;
  constructor(key: string) {
    this.#key = key;
  }
}
```

---

## 9. Class Types and Structural Typing

```typescript
// TypeScript uses STRUCTURAL typing for classes — not nominal
class Point {
  constructor(
    public x: number,
    public y: number,
  ) {}
}
class Coord {
  constructor(
    public x: number,
    public y: number,
  ) {}
}

let p: Point = new Coord(1, 2); // ✅ same structure!
let c: Coord = new Point(3, 4); // ✅

// This is unlike Java/C# where class identity matters
// In TS: if it has the right shape, it IS the type

// Working with constructor types (the type of a CLASS ITSELF, not instances)
type Constructor<T = {}> = new (...args: any[]) => T;
type AbstractConstructor<T = {}> = abstract new (...args: any[]) => T;

// Mixin pattern using constructor types
function Timestamped<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();

    touch() {
      this.updatedAt = new Date();
    }
  };
}

function Activatable<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    isActive = false;
    activate() {
      this.isActive = true;
    }
    deactivate() {
      this.isActive = false;
    }
  };
}

// Apply mixins
class User {
  constructor(public name: string) {}
}
const TimestampedUser = Timestamped(User);
const ActivatableUser = Activatable(TimestampedUser);

const user = new ActivatableUser("Alice");
user.name; // 'Alice'
user.createdAt; // Date
user.activate();
user.isActive; // true
```

---

## 10. Common Patterns

```typescript
// Pattern 1: Fluent builder
class QueryBuilder {
  private table?: string;
  private wheres: string[] = [];
  private limitN?: number;

  from(table: string): this {
    this.table = table;
    return this;
  }
  where(cond: string): this {
    this.wheres.push(cond);
    return this;
  }
  limit(n: number): this {
    this.limitN = n;
    return this;
  }

  build(): string {
    if (!this.table) throw new Error("Table is required");
    let sql = `SELECT * FROM ${this.table}`;
    if (this.wheres.length) sql += ` WHERE ${this.wheres.join(" AND ")}`;
    if (this.limitN) sql += ` LIMIT ${this.limitN}`;
    return sql;
  }
}

const sql = new QueryBuilder()
  .from("users")
  .where("active = 1")
  .limit(10)
  .build();

// Pattern 2: Abstract factory
abstract class DialogFactory {
  // Template method pattern
  createDialog(): void {
    const dialog = this.buildDialog(); // abstract — subclass provides
    dialog.render();
  }
  protected abstract buildDialog(): Dialog;
}
```

---

## 11. Common Mistakes

### Mistake 1 — Forgetting `super()` in derived class constructor

```typescript
class Animal {
  constructor(public name: string) {}
}

class Dog extends Animal {
  breed: string;
  constructor(name: string, breed: string) {
    // this.breed = breed; // ❌ Must call super() before accessing 'this'
    super(name); // ✅ super() first
    this.breed = breed;
  }
}
```

### Mistake 2 — Arrow methods vs prototype methods

```typescript
class Counter {
  count = 0;

  // Arrow: creates a NEW function PER INSTANCE (more memory, but `this` is always bound)
  increment = () => {
    this.count++;
  };

  // Prototype method: ONE function shared across all instances (less memory)
  decrement() {
    this.count--;
  }
}

// Arrow is safe to pass as a callback without binding:
const counter = new Counter();
setTimeout(counter.increment, 1000); // ✅ `this` is always the Counter instance
setTimeout(counter.decrement, 1000); // ❌ `this` might be wrong without bind
setTimeout(counter.decrement.bind(counter), 1000); // ✅ explicit bind
```

---

## 12. Exercises

### Exercise 1 — Generic repository base class

```typescript
// Implement an abstract generic Repository<T, ID> class with:
// - Abstract methods: findById, save, delete
// - Concrete method: findOrFail (throws if findById returns null)
// - Concrete method: saveAll (saves an array of items)
// Then implement a concrete InMemoryRepository<T extends { id: ID }, ID = string>
```

<details>
<summary>Solution</summary>

```typescript
abstract class Repository<T, ID = string> {
  abstract findById(id: ID): Promise<T | null>;
  abstract save(item: T): Promise<T>;
  abstract delete(id: ID): Promise<void>;

  async findOrFail(id: ID): Promise<T> {
    const item = await this.findById(id);
    if (item === null) throw new Error(`Item ${id} not found`);
    return item;
  }

  async saveAll(items: T[]): Promise<T[]> {
    return Promise.all(items.map((item) => this.save(item)));
  }
}

class InMemoryRepository<T extends { id: ID }, ID = string> extends Repository<
  T,
  ID
> {
  private store = new Map<ID, T>();

  async findById(id: ID): Promise<T | null> {
    return this.store.get(id) ?? null;
  }

  async save(item: T): Promise<T> {
    this.store.set(item.id, item);
    return item;
  }

  async delete(id: ID): Promise<void> {
    this.store.delete(id);
  }
}

const repo = new InMemoryRepository<User>();
await repo.save({ id: "1", name: "Alice", email: "alice@ex.com" });
await repo.findOrFail("1"); // User
await repo.findOrFail("2"); // throws
```

</details>

---

## 🔗 Related Topics

- [`07-generics.md`](./07-generics.md) — Generic classes and mixins
- [`03-interfaces-and-type-aliases.md`](./03-interfaces-and-type-aliases.md) — implements interface
- [`javascript-core/06-prototypes.md`](../javascript-core/06-prototypes.md) — How JS classes work under the hood
