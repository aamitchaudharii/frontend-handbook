# 16 — Variables and Data Types

> **"Before you can think in JavaScript, you need to know what JavaScript thinks a 'value' is — because it doesn't always agree with you. A string that looks like a number isn't a number. An empty array isn't false. And `null` somehow has type 'object'. Understanding types is understanding where JavaScript will surprise you."**

🟢 **Level: Beginner** — Start here if you're new to JavaScript.

JavaScript has two categories of values: **primitives** (simple, immutable, stored by value) and **objects** (complex, mutable, stored by reference). Every behavior that confuses beginners — equality bugs, unexpected mutation, mysterious type coercion — traces back to this distinction.

---

## 📚 Table of Contents

1. [Declaring Variables — var, let, const](#1-declaring-variables--var-let-const)
2. [Primitive Types](#2-primitive-types)
3. [Reference Types (Objects)](#3-reference-types-objects)
4. [typeof Operator](#4-typeof-operator)
5. [Type Coercion](#5-type-coercion)
6. [Equality — == vs ===](#6-equality----vs-)
7. [Falsy and Truthy Values](#7-falsy-and-truthy-values)
8. [Primitive vs Reference — The Critical Difference](#8-primitive-vs-reference--the-critical-difference)
9. [Good Practices](#9-good-practices)
10. [Bad Practices](#10-bad-practices)
11. [Common Mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)

---

## 1. Declaring Variables — var, let, const

```javascript
// var: function-scoped, hoisted, re-declarable (avoid in modern code)
var name = "Alice";
var name = "Bob"; // OK — re-declaration allowed (a footgun)

// let: block-scoped, not hoisted to value, re-assignable
let age = 25;
age = 26; // OK
// let age = 27; // ❌ SyntaxError — can't re-declare in same scope

// const: block-scoped, must be initialized, binding is immutable
const PI = 3.14159;
// PI = 3; // ❌ TypeError — can't reassign

// IMPORTANT: const doesn't make objects immutable — the BINDING is constant
const user = { name: "Alice" };
user.name = "Bob"; // ✅ works — you're mutating the object, not reassigning the variable
// user = {};      // ❌ TypeError — you CAN'T reassign the variable
```

### var vs let vs const at a glance

```
                  var         let         const
Scope:          function    block       block
Hoisted:        yes (undef) yes (TDZ)   yes (TDZ)
Re-declarable:  yes         no          no
Re-assignable:  yes         yes         no
Global property:yes (window) no         no

TDZ = Temporal Dead Zone: declared but not yet initialized
      (accessing before initialization → ReferenceError)
```

---

## 2. Primitive Types

JavaScript has **8 primitive types**. Primitives are immutable — you can't change the value itself, only reassign the variable.

```javascript
// 1. Number — all numbers are 64-bit floating point (IEEE 754)
let integer = 42;
let float = 3.14;
let negative = -7;
let infinity = Infinity;
let notANumber = NaN; // result of invalid math operations

// ⚠️ Floating point imprecision:
0.1 + 0.2 === 0.3; // false! (result is 0.30000000000000004)
Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON; // true — correct comparison

// Number limits:
Number.MAX_SAFE_INTEGER; // 9007199254740991 (2^53 - 1)
Number.MIN_SAFE_INTEGER; // -9007199254740991

// 2. BigInt — integers beyond Number.MAX_SAFE_INTEGER
const bigNumber = 9007199254740992n; // append n
const result = bigNumber + 1n; // 9007199254740993n

// 3. String — sequence of UTF-16 code units
let single = "hello";
let double = "world";
let template = `Hello ${single}`; // template literal: evaluates expressions

// Strings are immutable:
let str = "hello";
str[0] = "H"; // silently fails (no error, but str is still 'hello')
str = "Hello"; // ✅ this is a reassignment, not a mutation

// 4. Boolean
let yes = true;
let no = false;

// 5. undefined — declared but not yet assigned a value
let uninitialized; // undefined
function withoutReturn() {} // returns undefined

// 6. null — intentional absence of a value (must be explicitly set)
let empty = null;
// null !== undefined (different types, different intentions)

// 7. Symbol — unique, guaranteed-unique identifier (ES6)
const sym1 = Symbol("description");
const sym2 = Symbol("description");
sym1 === sym2; // false — every Symbol is unique
// Used for: object property keys that won't clash with other keys

// 8. BigInt — already shown above
```

---

## 3. Reference Types (Objects)

Everything that isn't a primitive is an **object** (stored by reference):

```javascript
// Object literal
const person = { name: "Alice", age: 30 };

// Array (a specialized object)
const colors = ["red", "green", "blue"];

// Function (a callable object)
function greet(name) {
  return `Hello, ${name}`;
}

// Date
const now = new Date();

// RegExp
const pattern = /^\d+$/;

// Map, Set, WeakMap, WeakSet
const map = new Map();
const set = new Set([1, 2, 3]);
```

---

## 4. typeof Operator

```javascript
typeof 42; // "number"
typeof 3.14; // "number"
typeof NaN; // "number" ← NaN is of type number!
typeof "hello"; // "string"
typeof true; // "boolean"
typeof undefined; // "undefined"
typeof Symbol(); // "symbol"
typeof 42n; // "bigint"
typeof {}; // "object"
typeof []; // "object" ← arrays are objects!
typeof null; // "object" ← historical bug, cannot be fixed
typeof function () {}; // "function" ← functions are callable objects

// To check for null specifically (can't use typeof):
value === null;

// To check if something is an array:
Array.isArray([]); // true ✅

// To check the type of any value accurately:
Object.prototype.toString.call([]); // "[object Array]"
Object.prototype.toString.call(null); // "[object Null]"
Object.prototype.toString.call(new Date()); // "[object Date]"
```

---

## 5. Type Coercion

JavaScript automatically converts values between types in certain operations — this is called **type coercion**.

```javascript
// String coercion: + with a string converts the other side to string
"5" + 3; // "53" (number 3 coerced to string "3")
"5" + true; // "5true"
"5" + null; // "5null"
"5" + undefined; // "5undefined"

// Numeric coercion: -, *, /, % always convert to number
"5" - 3; // 2 (string "5" coerced to number 5)
"5" * "2"; // 10
"5" - true; // 4 (true → 1)
"5" - false; // 5 (false → 0)
"5" - null; // 5 (null → 0)
"5" - undefined; // NaN (undefined → NaN)
"abc" - 1; // NaN ("abc" can't be numeric)

// Boolean coercion (in if conditions, ||, &&, !):
// Falsy values → false; everything else → true
Boolean(0); // false
Boolean(""); // false
Boolean(null); // false
Boolean(undefined); // false
Boolean(NaN); // false
Boolean(false); // false
Boolean(0n); // false
Boolean("0"); // true ← "0" is a non-empty string!
Boolean([]); // true ← empty array is truthy!
Boolean({}); // true ← empty object is truthy!

// Explicit coercion (always prefer this over relying on implicit):
Number("42"); // 42
Number(true); // 1
Number(false); // 0
Number(null); // 0
Number("abc"); // NaN

String(42); // "42"
String(true); // "true"
String(null); // "null"

Boolean(1); // true
Boolean(0); // false

parseInt("42px"); // 42 (stops at non-numeric character)
parseFloat("3.14"); // 3.14
+"42"; // 42 (unary + converts to number)
+true; // 1
```

---

## 6. Equality — == vs ===

```javascript
// === (strict equality): no coercion — must be same TYPE and VALUE
1 === 1; // true
1 === "1"; // false (different types)
null === null; // true
NaN === NaN; // false! (NaN is not equal to itself — use Number.isNaN())

// == (loose equality): performs type coercion before comparing
1 == "1"; // true ('1' → 1)
0 == false; // true (false → 0)
0 == ""; // true ('' → 0)
null == undefined; // true (special case)
null == 0; // false (null only == undefined with ==)
null == false; // false

// ✅ Rule of thumb: always use === unless you specifically need == null
// `value == null` catches BOTH null and undefined — the one valid use of ==
if (value == null) {
  /* value is null OR undefined */
}

// Comparing objects: == and === both check REFERENCE equality
const a = { x: 1 };
const b = { x: 1 };
a === b; // false — different objects in memory, even if contents are the same
const c = a;
a === c; // true — same reference
```

---

## 7. Falsy and Truthy Values

```javascript
// The 8 falsy values in JavaScript (everything else is truthy):
if (false) {
  /* never */
}
if (0) {
  /* never */
}
if (-0) {
  /* never */
}
if (0n) {
  /* never */
} // BigInt zero
if ("") {
  /* never */
}
if (null) {
  /* never */
}
if (undefined) {
  /* never */
}
if (NaN) {
  /* never */
}

// TRUTHY (may surprise you):
if ("0") {
  /* runs! */
} // non-empty string
if ("false") {
  /* runs! */
} // non-empty string
if ([]) {
  /* runs! */
} // empty array
if ({}) {
  /* runs! */
} // empty object
if (-1) {
  /* runs! */
} // any non-zero number

// Practical patterns using truthy/falsy:
const name = user.name || "Anonymous"; // fallback if name is falsy
const count = stats.count ?? 0; // ?? = nullish coalescing: only falls back for null/undefined
// (not for 0 or '' like || does)
```

---

## 8. Primitive vs Reference — The Critical Difference

This is the source of many beginner bugs:

```javascript
// PRIMITIVES: stored by VALUE — copies are independent
let a = 5;
let b = a; // b gets a COPY of the value 5
b = 10;
console.log(a); // 5 — a is unaffected

// OBJECTS: stored by REFERENCE — "copies" point to the same object
let obj1 = { x: 1 };
let obj2 = obj1; // obj2 gets a copy of the REFERENCE (they point to the same object)
obj2.x = 99;
console.log(obj1.x); // 99 ← obj1 was mutated!

// Same with arrays:
let arr1 = [1, 2, 3];
let arr2 = arr1;
arr2.push(4);
console.log(arr1); // [1, 2, 3, 4] ← arr1 was mutated!

// ✅ Creating a true copy (shallow):
let obj3 = { ...obj1 }; // spread operator
let arr3 = [...arr1]; // spread operator
let obj4 = Object.assign({}, obj1);

// ✅ Deep copy (handles nested objects):
let obj5 = structuredClone(obj1); // modern browsers/Node.js 17+
let obj6 = JSON.parse(JSON.stringify(obj1)); // older but limited
```

---

## 9. Good Practices

### ✅ Prefer const by default, use let when reassignment is needed

```javascript
// ✅ Signal intent clearly
const MAX_RETRIES = 3;
const users = []; // const: but the array can still be mutated (push, etc.)
users.push({ name: "Alice" }); // this is fine

let currentUser = null; // let: will be reassigned
currentUser = { name: "Bob" };
```

### ✅ Use === for all equality checks

```javascript
// ✅ Explicit and predictable
if (value === null || value === undefined) {
  /* handle missing value */
}
// or equivalently (the ONE valid use of ==):
if (value == null) {
  /* same thing */
}
```

### ✅ Use Number.isNaN() not the global isNaN()

```javascript
isNaN("hello"); // true — coerces 'hello' to NaN first (misleading)
Number.isNaN("hello"); // false — 'hello' is not NaN, it's a string ✅
Number.isNaN(NaN); // true ✅
```

---

## 10. Bad Practices

### ❌ Using var (in modern code)

```javascript
// ❌ var is function-scoped and can lead to confusing bugs
function example() {
  if (true) {
    var x = 5;
  } // x leaks to the function scope
  console.log(x); // 5 — surprising if you expected block scope
}
// ✅ Use let or const: they're block-scoped
```

### ❌ Relying on implicit type coercion for equality

```javascript
// ❌ Unexpected coercion
if (user.age == "18") {
  /* might work, but fragile */
}
// ✅ Be explicit
if (user.age === 18) {
  /* clear intent */
}
if (Number(user.age) === 18) {
  /* if you need to coerce, do it explicitly */
}
```

---

## 11. Common Mistakes

### Mistake 1 — Thinking empty array/object are falsy

```javascript
// ❌ This check is wrong
if (![] || !{}) {
  // never runs — [] and {} are TRUTHY
}

// ✅ Check what you actually mean
if (array.length === 0) {
  /* empty array */
}
if (Object.keys(obj).length === 0) {
  /* empty object */
}
```

### Mistake 2 — Mutating through a const reference

```javascript
// ❌ Confusing: const doesn't prevent mutation
const settings = { theme: "dark" };
settings.theme = "light"; // This WORKS — const only prevents reassignment

// If you need a truly immutable object:
const frozen = Object.freeze({ theme: "dark" });
frozen.theme = "light"; // Silently fails (or throws in strict mode)
frozen.theme; // still 'dark'
```

### Mistake 3 — NaN comparisons

```javascript
const result = parseInt("abc"); // NaN
if (result === NaN) {
  /* never runs — NaN !== NaN */
}
if (Number.isNaN(result)) {
  /* ✅ correct */
}
```

---

## 12. Exercises

### Exercise 1 — Predict the output

```javascript
// What does each line print?
console.log(typeof null);
console.log(typeof []);
console.log(typeof function () {});
console.log(1 + "2");
console.log("5" - 3);
console.log(null == undefined);
console.log(null === undefined);
console.log(Boolean([]));
console.log(Boolean(""));
```

<details>
<summary>Solution</summary>

```javascript
console.log(typeof null); // "object" (historical bug)
console.log(typeof []); // "object" (arrays are objects)
console.log(typeof function () {}); // "function"
console.log(1 + "2"); // "12" (1 coerced to "1", concatenated)
console.log("5" - 3); // 2 ("5" coerced to 5)
console.log(null == undefined); // true (special loose equality rule)
console.log(null === undefined); // false (different types)
console.log(Boolean([])); // true (empty array is truthy)
console.log(Boolean("")); // false (empty string is falsy)
```

</details>

---

## 🔗 Related Topics

- [`17-operators-and-expressions.md`](./17-operators-and-expressions.md) — Operators that work on these types
- [`07-scope-chain.md`](./07-scope-chain.md) — How var/let/const affect scope
- [`05-closures.md`](./05-closures.md) — How values are captured in closures
