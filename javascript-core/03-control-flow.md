# 18 — Control Flow

> **"Control flow is how a program makes decisions. Mastering it isn't just about knowing the syntax — it's about choosing the right structure for the decision being made, so the code reads like the intent behind it."**

🟢 **Level: Beginner**

---

## 📚 Table of Contents

1. [if / else if / else](#1-if--else-if--else)
2. [switch Statement](#2-switch-statement)
3. [for Loop](#3-for-loop)
4. [while and do…while](#4-while-and-dowhile)
5. [for…of — Iterating Values](#5-forof--iterating-values)
6. [for…in — Iterating Keys](#6-forin--iterating-keys)
7. [break and continue](#7-break-and-continue)
8. [Labeled Statements](#8-labeled-statements)
9. [Early Return Pattern](#9-early-return-pattern)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. if / else if / else

```javascript
const score = 75;

if (score >= 90) {
  console.log("A");
} else if (score >= 80) {
  console.log("B");
} else if (score >= 70) {
  console.log("C");
} else {
  console.log("F");
}
// "C"

// Single-line (no braces) — legal but discouraged
if (score >= 70) console.log("Passing");

// Always use braces {} even for single statements — avoids bugs when adding lines
if (score >= 70) {
  console.log("Passing"); // ✅ clear scope
}
```

---

## 2. switch Statement

```javascript
const day = "Monday";

switch (day) {
  case "Monday":
  case "Tuesday":
  case "Wednesday":
  case "Thursday":
  case "Friday":
    console.log("Weekday");
    break; // CRITICAL: without break, execution FALLS THROUGH to the next case!

  case "Saturday":
  case "Sunday":
    console.log("Weekend");
    break;

  default:
    console.log("Unknown day");
}

// switch uses === for comparison (strict, no coercion)
// FALL-THROUGH bug example:
switch (status) {
  case "active":
    console.log("Active"); // if status is 'active' AND there's no break:
  case "pending":
    console.log("Pending"); // THIS ALSO RUNS for status === 'active'!
    break;
}
// ✅ Unless intentional (multiple cases sharing code), always break or return

// Modern alternative: object lookup table (often cleaner)
const grades = { A: 90, B: 80, C: 70 };
const minScore = grades[letter]; // O(1) lookup, no switch needed
```

---

## 3. for Loop

```javascript
// Classic for loop
for (let i = 0; i < 5; i++) {
  console.log(i); // 0, 1, 2, 3, 4
}

// Iterating an array by index
const fruits = ["apple", "banana", "cherry"];
for (let i = 0; i < fruits.length; i++) {
  console.log(fruits[i]);
}

// Iterating in reverse
for (let i = fruits.length - 1; i >= 0; i--) {
  console.log(fruits[i]); // 'cherry', 'banana', 'apple'
}

// Every part is optional (infinite loop):
// for (;;) { break; } // ← valid, but avoid — use while (true) for clarity

// Nested loops
for (let row = 0; row < 3; row++) {
  for (let col = 0; col < 3; col++) {
    console.log(`[${row}][${col}]`);
  }
}
```

---

## 4. while and do…while

```javascript
// while: condition checked BEFORE each iteration
let count = 0;
while (count < 5) {
  console.log(count);
  count++;
}

// do...while: condition checked AFTER each iteration (body always runs at least once)
let input;
do {
  input = getInput(); // runs at least once, even if input is already valid
} while (!isValid(input));

// when to use while vs for:
// for: when you know the number of iterations in advance (or are iterating a collection)
// while: when the loop continues until a condition changes (e.g., retry until success)
```

---

## 5. for…of — Iterating Values

```javascript
// for...of works on any ITERABLE: arrays, strings, Map, Set, generators
const colors = ["red", "green", "blue"];

for (const color of colors) {
  console.log(color); // 'red', 'green', 'blue'
}

// With index via entries()
for (const [index, color] of colors.entries()) {
  console.log(`${index}: ${color}`); // "0: red", "1: green", "2: blue"
}

// Iterating a string (character by character)
for (const char of "hello") {
  console.log(char); // 'h', 'e', 'l', 'l', 'o'
}

// Iterating a Map
const map = new Map([
  ["a", 1],
  ["b", 2],
]);
for (const [key, value] of map) {
  console.log(key, value); // 'a' 1, 'b' 2
}

// Iterating a Set
const set = new Set([1, 2, 3]);
for (const value of set) {
  console.log(value); // 1, 2, 3
}
```

---

## 6. for…in — Iterating Keys

```javascript
// for...in iterates the ENUMERABLE PROPERTY NAMES of an object
const person = { name: "Alice", age: 30, city: "NY" };
for (const key in person) {
  console.log(key, person[key]); // 'name' 'Alice', 'age' 30, 'city' 'NY'
}

// ⚠️ for...in also includes INHERITED properties (from the prototype chain)
function Animal(name) {
  this.name = name;
}
Animal.prototype.type = "animal";
const cat = new Animal("Whiskers");
for (const key in cat) {
  console.log(key); // 'name' AND 'type' (inherited from prototype!)
}

// ✅ Guard against inherited properties:
for (const key in cat) {
  if (Object.prototype.hasOwnProperty.call(cat, key)) {
    console.log(key); // only 'name'
  }
}

// ⚠️ for...in is NOT reliable for arrays (don't use it for arrays):
const arr = [10, 20, 30];
for (const key in arr) {
  console.log(key); // '0', '1', '2' — string keys, not numeric
  // Also includes any custom properties added to Array.prototype
}
// ✅ Use for...of for arrays
```

---

## 7. break and continue

```javascript
// break: exits the ENTIRE loop immediately
for (let i = 0; i < 10; i++) {
  if (i === 5) break;
  console.log(i); // 0, 1, 2, 3, 4
}

// continue: skips the CURRENT iteration, moves to the next
for (let i = 0; i < 10; i++) {
  if (i % 2 === 0) continue; // skip even numbers
  console.log(i); // 1, 3, 5, 7, 9
}

// break in switch (see section 2)
// break in while
let value = prompt("Enter a positive number:");
while (true) {
  if (Number(value) > 0) break;
  value = prompt("Try again:");
}
```

---

## 8. Labeled Statements

```javascript
// Labels allow break/continue to target an OUTER loop (rarely needed)
outer: for (let i = 0; i < 3; i++) {
  inner: for (let j = 0; j < 3; j++) {
    if (j === 1) continue outer; // skip rest of outer loop's current iteration
    if (i === 1) break outer; // break ALL loops
    console.log(i, j);
  }
}

// In practice: labeled breaks are uncommon — prefer refactoring into functions
function findPair(matrix, target) {
  for (let i = 0; i < matrix.length; i++) {
    for (let j = 0; j < matrix[i].length; j++) {
      if (matrix[i][j] === target) {
        return [i, j]; // early return achieves the same as labeled break
      }
    }
  }
  return null;
}
```

---

## 9. Early Return Pattern

```javascript
// ❌ Deeply nested if/else (hard to read)
function processOrder(order) {
  if (order) {
    if (order.isValid) {
      if (order.items.length > 0) {
        // actual logic buried three levels deep
        return calculateTotal(order);
      } else {
        return "Empty order";
      }
    } else {
      return "Invalid order";
    }
  } else {
    return "No order";
  }
}

// ✅ Early return / guard clause pattern (flat, readable)
function processOrder(order) {
  if (!order) return "No order";
  if (!order.isValid) return "Invalid order";
  if (!order.items.length) return "Empty order";

  // The "happy path" is now unindented and obvious
  return calculateTotal(order);
}
```

---

## 10. Common Mistakes

### Mistake 1 — Off-by-one errors in loops

```javascript
// ❌ Accesses index 5 on a 5-element array (undefined)
const arr = [1, 2, 3, 4, 5];
for (let i = 0; i <= arr.length; i++) {
  // should be i < arr.length
  console.log(arr[i]); // last iteration: arr[5] = undefined
}

// ✅
for (let i = 0; i < arr.length; i++) {
  /* ... */
}
```

### Mistake 2 — Forgetting break in switch

```javascript
// ❌ Fall-through bug
switch (status) {
  case "active":
    activate();
  // forgot break — FALLS THROUGH to 'pending'
  case "pending":
    setPending(); // ALSO called when status is 'active' — unintentional!
    break;
}
```

### Mistake 3 — Modifying an array while iterating with index

```javascript
// ❌ Skips elements when removing items
const arr = [1, 2, 3, 4, 5];
for (let i = 0; i < arr.length; i++) {
  if (arr[i] % 2 === 0) arr.splice(i, 1); // removes element, shifts all indices
  // next iteration checks the same index, which now has the NEXT element
}

// ✅ Iterate backwards when removing by index
for (let i = arr.length - 1; i >= 0; i--) {
  if (arr[i] % 2 === 0) arr.splice(i, 1);
}

// ✅ Or use filter (preferred — creates a new array, no mutation)
const odds = arr.filter((n) => n % 2 !== 0);
```

---

## 11. Exercises

### Exercise 1 — FizzBuzz

```javascript
// Print numbers 1-30. For multiples of 3, print "Fizz".
// For multiples of 5, print "Buzz". For multiples of both, print "FizzBuzz".
```

<details>
<summary>Solution</summary>

```javascript
for (let i = 1; i <= 30; i++) {
  if (i % 15 === 0) console.log("FizzBuzz");
  else if (i % 3 === 0) console.log("Fizz");
  else if (i % 5 === 0) console.log("Buzz");
  else console.log(i);
}
```

</details>

### Exercise 2 — Find first duplicate

```javascript
// Given an array of numbers, return the first value that appears more than once.
// findFirstDuplicate([4, 1, 3, 2, 2, 3]) → 2
```

<details>
<summary>Solution</summary>

```javascript
function findFirstDuplicate(arr) {
  const seen = new Set();
  for (const num of arr) {
    if (seen.has(num)) return num;
    seen.add(num);
  }
  return null;
}
```

</details>

---

## 🔗 Related Topics

- [`17-operators-and-expressions.md`](./17-operators-and-expressions.md) — Operators used in conditions
- [`19-functions-fundamentals.md`](./19-functions-fundamentals.md) — Early return patterns inside functions
- [`20-arrays-and-iteration.md`](./20-arrays-and-iteration.md) — Higher-order iteration methods (map/filter/reduce)
