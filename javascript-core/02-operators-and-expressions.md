# 17 — Operators and Expressions

> **"An expression is anything that produces a value. An operator is what transforms or combines values to produce new ones. Learning operators isn't about memorizing a table — it's about understanding which ones can surprise you, and why."**

🟢 **Level: Beginner**

---

## 📚 Table of Contents

1. [Arithmetic Operators](#1-arithmetic-operators)
2. [Assignment Operators](#2-assignment-operators)
3. [Comparison Operators](#3-comparison-operators)
4. [Logical Operators](#4-logical-operators)
5. [Nullish Coalescing and Optional Chaining](#5-nullish-coalescing-and-optional-chaining)
6. [Bitwise Operators](#6-bitwise-operators)
7. [Ternary Operator](#7-ternary-operator)
8. [Operator Precedence](#8-operator-precedence)
9. [Common Mistakes](#9-common-mistakes)
10. [Exercises](#10-exercises)

---

## 1. Arithmetic Operators

```javascript
// Basic arithmetic
5 + 3; // 8  — addition
5 - 3; // 2  — subtraction
5 * 3; // 15 — multiplication
5 / 2; // 2.5 — division (always floating point)
5 % 2; // 1  — remainder (modulo)
5 ** 2; // 25 — exponentiation (ES2016)

// Integer division (JavaScript has no integer division operator)
Math.floor(5 / 2); // 2 — truncate toward negative infinity
Math.trunc(5 / 2); // 2 — truncate toward zero (handles negatives differently)
Math.trunc(-5 / 2); // -2 (vs Math.floor(-5/2) = -3)

// Increment / decrement
let x = 5;
x++; // post-increment: returns 5, THEN increments x to 6
++x; // pre-increment:  increments x first, THEN returns value
x--; // post-decrement
--x; // pre-decrement

// Unary plus / minus (type conversion)
+"42"; // 42  — converts string to number
+true; // 1
+false; // 0
+null; // 0
+undefined; // NaN

-"42"; // -42 — converts and negates

// String concatenation (+ with strings)
"Hello" + " " + "World"; // "Hello World"
"Value: " + 42; // "Value: 42" (42 coerced to string)
```

---

## 2. Assignment Operators

```javascript
let x = 10; // simple assignment

// Compound assignment (shorthand)
x += 5; // x = x + 5   → 15
x -= 3; // x = x - 3   → 12
x *= 2; // x = x * 2   → 24
x /= 4; // x = x / 4   → 6
x %= 4; // x = x % 4   → 2
x **= 3; // x = x ** 3  → 8

// Logical assignment (ES2021)
let a = null;
a ||= "default"; // a = a || 'default'  → 'default' (assigns if a is falsy)

let b = 0;
b ||= "default"; // 'default' (0 is falsy)
b ??= "default"; // 'default' ONLY if b is null/undefined (0 stays 0)

let c = 5;
c &&= c * 2; // c = c && c * 2 → 10 (assigns if c is truthy)

// Destructuring assignment
const { name, age } = { name: "Alice", age: 30 };
const [first, second, ...rest] = [1, 2, 3, 4, 5];
// first = 1, second = 2, rest = [3, 4, 5]
```

---

## 3. Comparison Operators

```javascript
// Strict equality (always prefer these)
5 === 5; // true
5 === "5"; // false — different types
null === null; // true

// Strict inequality
5 !== 5; // false
5 !== "5"; // true

// Relational (these DO coerce to numbers)
5 > 3; // true
5 >= 5; // true
5 < 3; // false
5 <= 4; // false

// String comparison (lexicographic — not numeric)
"banana" > "apple"; // true (b > a alphabetically)
"10" > "9"; // false! ('1' < '9' as strings)
10 > 9; // true  (numeric comparison)

// ✅ Always compare numbers as numbers, not as strings
const userInput = "10";
parseInt(userInput, 10) > 9; // true — explicit conversion
```

---

## 4. Logical Operators

```javascript
// && — returns the FIRST falsy value, or the LAST value if all are truthy
true && "hello"; // 'hello'
false && "hello"; // false
null && "hello"; // null
0 && "hello"; // 0
"yes" && "hello"; // 'hello' (both truthy, returns last)

// Practical: guard condition before evaluating
const name = user && user.name; // safe: only accesses .name if user is truthy

// || — returns the FIRST truthy value, or the LAST value if all are falsy
false || "default"; // 'default'
null || "default"; // 'default'
0 || "default"; // 'default'
"Alice" || "default"; // 'Alice'  (first truthy value)
false || null || 0; // 0 (all falsy, returns last)

// Practical: default value pattern
const displayName = user.name || "Anonymous";
// ⚠️ PROBLEM: 0 and '' are falsy too — what if a valid value is 0?
// ✅ Use ?? instead (nullish coalescing, see Section 5)

// ! — logical NOT (coerces to boolean and inverts)
!true; // false
!false; // true
!null; // true  (null is falsy, so !null is true)
!"hello"; // false ('hello' is truthy, so !'hello' is false)
!0; // true

// !! — double negation: coerce ANY value to a boolean
!!"hello"; // true
!!null; // false
!!0; // false
!![]; // true
```

---

## 5. Nullish Coalescing and Optional Chaining

```javascript
// ?? — nullish coalescing: fallback ONLY for null/undefined (not for 0 or '')
const count = response.count ?? 0;
// If count is 0, it stays 0 (0 is a valid value, not "missing")
// If count is null or undefined, it becomes 0
// Contrast with ||: response.count || 0 would ALSO replace 0 with 0 (fine here
//   but wrong if 0 means something different from "absent")

const port = config.port ?? 3000;
// 0 would be kept; null/undefined falls back to 3000

// ?. — optional chaining: short-circuits to undefined if LHS is null/undefined
const city = user?.address?.city; // undefined if user or address is null/undefined
// no TypeError thrown

const firstTag = post?.tags?.[0]; // array access
const result = obj?.method?.(); // method call

// Practical combinations
const userName = user?.profile?.name ?? "Guest";
// → Get deeply nested value safely, fall back to 'Guest' if anything is missing

// WITHOUT optional chaining (the old way — error-prone):
const cityOld = user && user.address && user.address.city;
```

---

## 6. Bitwise Operators

Used in low-level code, algorithms, and sometimes performance-critical paths. Operate on 32-bit integers.

```javascript
// Bitwise AND, OR, XOR, NOT
5 & 3; // 1  — 0101 & 0011 = 0001
5 | 3; // 7  — 0101 | 0011 = 0111
5 ^ 3; // 6  — 0101 ^ 0011 = 0110 (XOR: 1 if bits differ)
~5; // -6 — inverts bits (two's complement)

// Bit shifts
5 << 1; // 10 — shift left = multiply by 2
5 >> 1; // 2  — shift right (signed) = divide by 2
5 >>> 1; // 2  — unsigned right shift (fills with 0)

// Practical use: fast floor division
const half = value >> 1; // same as Math.floor(value / 2) for positive integers
const idx = n >>> 0; // convert to unsigned 32-bit integer (common in algorithms)
```

---

## 7. Ternary Operator

```javascript
// Syntax: condition ? valueIfTrue : valueIfFalse
const age = 20;
const status = age >= 18 ? "adult" : "minor"; // 'adult'

// Inline rendering in JSX (React)
const element = <div>{isLoading ? <Spinner /> : <Content />}</div>;

// Nested ternary (generally discouraged — use if/else instead)
const grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : "F";
// ✅ Better written as:
function getGrade(score) {
  if (score >= 90) return "A";
  if (score >= 80) return "B";
  if (score >= 70) return "C";
  return "F";
}
```

---

## 8. Operator Precedence

Higher precedence = evaluated first. When in doubt, use parentheses.

```javascript
// Common precedence order (highest to lowest in this list):
// 1. () — grouping
// 2. . [] ?.  — member access, optional chaining
// 3. ++ -- (postfix)
// 4. ! ~ + - (unary), ++ -- (prefix)
// 5. **  — exponentiation (right-to-left associative)
// 6. * / % — multiplication, division, modulo
// 7. + - — addition, subtraction
// 8. << >> >>> — bit shifts
// 9. < <= > >= — relational
// 10. === !== == != — equality
// 11. & — bitwise AND
// 12. ^ — bitwise XOR
// 13. | — bitwise OR
// 14. && — logical AND
// 15. || — logical OR
// 16. ?? — nullish coalescing
// 17. ?: — ternary
// 18. = += -= etc. — assignment

// EXAMPLES of surprising precedence:
2 + 3 * 4; // 14, not 20 (* before +)
(2 + 3) * 4; // 20 — parentheses override

true || (false && false); // true (&&  before ||)
(true || false) && false; // false

2 ** (3 ** 2); // 512, not 64 (** is RIGHT-to-left: 2 ** (3**2) = 2**9)

// When mixing && and ||, ALWAYS use parentheses:
const a = true,
  b = false,
  c = true;
a || (b && c); // true — b && c first, then a || result
(a || b) && c; // true — different evaluation
```

---

## 9. Common Mistakes

### Mistake 1 — Using = in a condition (assignment instead of comparison)

```javascript
// ❌ Assigns 5 to x inside the condition (always truthy)
let x = 0;
if ((x = 5)) {
  console.log(x); // 5 — and the condition was true because 5 is truthy
}

// ✅ Compare with ===
if (x === 5) {
  /* ... */
}
```

### Mistake 2 — || for defaults when 0 is a valid value

```javascript
// ❌ 0 is falsy — falls back to 10 even though 0 is a valid count
const count = options.count || 10;
// If options.count = 0 → count becomes 10 (wrong!)

// ✅ Use ?? which only triggers on null/undefined
const count = options.count ?? 10;
// If options.count = 0 → count stays 0 ✅
// If options.count = null → count becomes 10 ✅
```

### Mistake 3 — Confusing && short-circuit return with boolean

```javascript
// ❌ Expecting a boolean
const isValid = value && validate(value);
// If value is truthy: isValid = result of validate(value) (may be truthy/falsy)
// If value is falsy: isValid = the falsy value itself (not false)

// ✅ Explicitly convert to boolean if you need a boolean
const isValid = Boolean(value && validate(value));
// or
const isValid = !!(value && validate(value));
```

---

## 10. Exercises

### Exercise 1 — Predict the output

```javascript
console.log(2 + "3");
console.log("5" - 2);
console.log(null ?? "fallback");
console.log(0 ?? "fallback");
console.log(0 || "fallback");
console.log(false && "hello");
console.log(false || "hello");
console.log(null?.name);
console.log(null?.name ?? "unknown");
```

<details>
<summary>Solution</summary>

```javascript
console.log(2 + "3"); // "23" (number coerced to string)
console.log("5" - 2); // 3 (string coerced to number)
console.log(null ?? "fallback"); // "fallback" (null triggers ??)
console.log(0 ?? "fallback"); // 0 (0 is not null/undefined)
console.log(0 || "fallback"); // "fallback" (0 is falsy, triggers ||)
console.log(false && "hello"); // false (first falsy value)
console.log(false || "hello"); // "hello" (first truthy value)
console.log(null?.name); // undefined (optional chain short-circuits)
console.log(null?.name ?? "unknown"); // "unknown" (undefined triggers ??)
```

</details>

---

## 🔗 Related Topics

- [`16-variables-and-data-types.md`](./16-variables-and-data-types.md) — Type coercion affects operator behavior
- [`18-control-flow.md`](./18-control-flow.md) — Using operators in conditions
