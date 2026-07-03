# 20 — Arrays and Iteration

> **"Arrays are the workhorse of JavaScript data manipulation. The entire ecosystem of functional programming in JS — map, filter, reduce — lives here. Master these methods and you replace 80% of your manual loops with expressive, composable one-liners."**

🟢 **Level: Beginner → Intermediate**

---

## 📚 Table of Contents

1. [Creating Arrays](#1-creating-arrays)
2. [Reading and Mutating](#2-reading-and-mutating)
3. [Array Spread and Destructuring](#3-array-spread-and-destructuring)
4. [Iteration Methods — The Big Six](#4-iteration-methods--the-big-six)
5. [Search Methods](#5-search-methods)
6. [Sorting and Reversing](#6-sorting-and-reversing)
7. [Flattening and Combining](#7-flattening-and-combining)
8. [Useful Static Methods](#8-useful-static-methods)
9. [Common Patterns](#9-common-patterns)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. Creating Arrays

```javascript
// Literal (preferred)
const fruits = ["apple", "banana", "cherry"];
const empty = [];
const mixed = [1, "two", true, null, { x: 3 }];

// Array constructor (avoid for single numbers)
new Array(3); // [empty × 3] — a sparse array of length 3 (trap!)
new Array(1, 2, 3); // [1, 2, 3]
Array.from({ length: 3 }, (_, i) => i); // [0, 1, 2] — safe alternative

// Array.from: convert any iterable or array-like to an array
Array.from("hello"); // ['h', 'e', 'l', 'l', 'o']
Array.from(new Set([1, 2, 2, 3])); // [1, 2, 3]
Array.from(document.querySelectorAll("li")); // NodeList → real array
Array.from({ length: 5 }, (_, i) => i * 2); // [0, 2, 4, 6, 8]

// Array.of: create array from arguments (unlike new Array())
Array.of(3); // [3] — NOT a sparse array of length 3
Array.of(1, 2); // [1, 2]
```

---

## 2. Reading and Mutating

```javascript
const arr = ["a", "b", "c", "d"];

// Reading
arr[0]; // 'a'
arr[arr.length - 1]; // 'd' — last element
arr.at(-1); // 'd' — modern: negative index from end (ES2022)
arr.at(-2); // 'c'

// MUTATING methods (modify the original array):
arr.push("e"); // adds to END;    returns new length → ['a','b','c','d','e']
arr.pop(); // removes from END; returns removed element → ['a','b','c','d']
arr.unshift("z"); // adds to START;  returns new length → ['z','a','b','c','d']
arr.shift(); // removes from START; returns removed element → ['a','b','c','d']

arr.splice(1, 2); // remove 2 elements at index 1 → returns ['b','c'], arr = ['a','d']
arr.splice(1, 0, "X", "Y"); // insert at index 1 without removing
arr.splice(1, 1, "Z"); // replace element at index 1

arr.reverse(); // reverses IN PLACE, returns the array
arr.sort(); // sorts IN PLACE (default: string comparison!)
arr.fill(0, 1, 3); // fill indices 1-2 with 0 (excludes end index)

// NON-MUTATING methods (return new array/value, original unchanged):
arr.slice(1, 3); // ['b', 'c'] — new array from index 1 up to (not including) 3
arr.concat([4, 5]); // new array joining this + argument
arr.join(" - "); // 'a - b - c - d' — convert to string with separator
```

---

## 3. Array Spread and Destructuring

```javascript
// Spread: expand array into individual elements
const a = [1, 2, 3];
const b = [4, 5, 6];
const combined = [...a, ...b]; // [1, 2, 3, 4, 5, 6]
const copy = [...a]; // shallow copy
const withExtra = [...a, 99]; // [1, 2, 3, 99]

Math.max(...a); // 3 — spread into function arguments

// Destructuring: extract array elements into variables
const [first, second, third] = [10, 20, 30];
// first = 10, second = 20, third = 30

// Skip elements
const [, , last] = [10, 20, 30]; // last = 30

// Rest element
const [head, ...tail] = [1, 2, 3, 4, 5];
// head = 1, tail = [2, 3, 4, 5]

// Default values
const [x = 0, y = 0, z = 0] = [1, 2];
// x = 1, y = 2, z = 0 (default)

// Swap variables
let p = 1,
  q = 2;
[p, q] = [q, p]; // p = 2, q = 1 — no temp variable needed

// From a function returning an array
function getCoords() {
  return [51.5, -0.1];
}
const [lat, lng] = getCoords();
```

---

## 4. Iteration Methods — The Big Six

```javascript
const numbers = [1, 2, 3, 4, 5];

// forEach: iterate for side effects (returns undefined)
numbers.forEach((num, index, array) => {
  console.log(num);
});
// Cannot break out of forEach — use for...of instead

// map: transform each element, returns NEW array of SAME length
const doubled = numbers.map((n) => n * 2); // [2, 4, 6, 8, 10]
const strings = numbers.map((n) => `${n}!`); // ['1!', '2!', '3!', '4!', '5!']

// filter: keep elements where callback returns true, returns NEW (shorter) array
const evens = numbers.filter((n) => n % 2 === 0); // [2, 4]
const large = numbers.filter((n) => n > 3); // [4, 5]

// reduce: accumulate elements into a SINGLE value
const sum = numbers.reduce((acc, n) => acc + n, 0); // 15
const product = numbers.reduce((acc, n) => acc * n, 1); // 120
// Second argument to reduce is the INITIAL VALUE — always provide it!

// Reducing to an object (grouping)
const words = ["apple", "ant", "banana", "blueberry", "avocado"];
const byLetter = words.reduce((acc, word) => {
  const letter = word[0];
  acc[letter] = acc[letter] ? [...acc[letter], word] : [word];
  return acc;
}, {});
// { a: ['apple', 'ant', 'avocado'], b: ['banana', 'blueberry'] }

// some: returns true if AT LEAST ONE element passes the test
const hasEven = numbers.some((n) => n % 2 === 0); // true

// every: returns true if ALL elements pass the test
const allPositive = numbers.every((n) => n > 0); // true
const allEven = numbers.every((n) => n % 2 === 0); // false

// Chaining (common pattern)
const result = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  .filter((n) => n % 2 === 0) // [2, 4, 6, 8, 10]
  .map((n) => n * n) // [4, 16, 36, 64, 100]
  .reduce((sum, n) => sum + n, 0); // 220
```

---

## 5. Search Methods

```javascript
const arr = [10, 20, 30, 20, 40];

// find: returns the FIRST element that passes the test (or undefined)
arr.find((n) => n > 15); // 20

// findIndex: returns the INDEX of the first match (or -1)
arr.findIndex((n) => n > 15); // 1

// findLast / findLastIndex (ES2023): search from the end
arr.findLast((n) => n === 20); // 20 (the second 20)
arr.findLastIndex((n) => n === 20); // 3

// indexOf: finds a value (strict equality), returns first index or -1
arr.indexOf(20); // 1 (first occurrence)
arr.lastIndexOf(20); // 3 (last occurrence)
arr.indexOf(99); // -1 (not found)

// includes: checks for existence (handles NaN correctly, unlike indexOf)
arr.includes(30); // true
arr.includes(99); // false
[1, NaN, 3].includes(NaN); // true
[1, NaN, 3].indexOf(NaN); // -1 (indexOf can't find NaN)
```

---

## 6. Sorting and Reversing

```javascript
// sort() — MUTATES the array, default sorts as STRINGS (trap!)
[10, 2, 1, 20].sort(); // [1, 10, 2, 20] ← lexicographic, not numeric!
[10, 2, 1, 20].sort((a, b) => a - b); // [1, 2, 10, 20] ← numeric ascending
[10, 2, 1, 20].sort((a, b) => b - a); // [20, 10, 2, 1] ← numeric descending

// Sort strings (locale-aware)
["banana", "Apple", "cherry"].sort(); // ['Apple', 'banana', 'cherry'] (uppercase first)
["banana", "Apple", "cherry"].sort((a, b) => a.localeCompare(b)); // locale-aware

// Sort objects by property
const users = [
  { name: "Charlie", age: 30 },
  { name: "Alice", age: 25 },
  { name: "Bob", age: 35 },
];
users.sort((a, b) => a.age - b.age); // by age ascending
users.sort((a, b) => a.name.localeCompare(b.name)); // by name alphabetically

// Non-mutating sort (create a sorted copy first)
const sortedCopy = [...arr].sort((a, b) => a - b); // original arr unchanged

// toSorted() — non-mutating (ES2023)
const sorted = arr.toSorted((a, b) => a - b); // new array, original unchanged

// reverse() — mutates, toReversed() — non-mutating (ES2023)
arr.reverse(); // mutates
const rev = arr.toReversed(); // new array (ES2023)
```

---

## 7. Flattening and Combining

```javascript
// flat: flatten nested arrays (depth defaults to 1)
[1, [2, [3, [4]]]].flat(); // [1, 2, [3, [4]]] — 1 level deep
[1, [2, [3, [4]]]].flat(2); // [1, 2, 3, [4]] — 2 levels
[1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4] — fully flatten

// flatMap: map then flatten 1 level (more efficient than map + flat)
["hello world", "foo bar"].flatMap((s) => s.split(" ")); // ['hello', 'world', 'foo', 'bar']
[1, 2, 3].flatMap((n) => [n, n * 2]); // [1, 2, 2, 4, 3, 6]

// concat: combine arrays
[1, 2].concat([3, 4], [5, 6]); // [1, 2, 3, 4, 5, 6]
[1, 2].concat(3, 4); // [1, 2, 3, 4]

// toSpliced (ES2023): non-mutating splice
const arr = [1, 2, 3, 4];
const result = arr.toSpliced(1, 2, "a", "b"); // [1, 'a', 'b', 4]
// arr is still [1, 2, 3, 4]

// with (ES2023): non-mutating index update
const updated = arr.with(2, 99); // [1, 2, 99, 4]
// arr unchanged
```

---

## 8. Useful Static Methods

```javascript
// Array.isArray: reliable type check (typeof returns 'object' for arrays)
Array.isArray([]); // true
Array.isArray({}); // false
Array.isArray("hello"); // false

// Array.from with mapping
Array.from({ length: 5 }, (_, i) => i + 1); // [1, 2, 3, 4, 5]
Array.from("hello", (c) => c.toUpperCase()); // ['H', 'E', 'L', 'L', 'O']

// Array.of (already covered)
Array.of(1, 2, 3); // [1, 2, 3]
```

---

## 9. Common Patterns

```javascript
// Remove duplicates
const deduped = [...new Set([1, 2, 2, 3, 3, 3])]; // [1, 2, 3]

// Chunk array into groups of size n
function chunk(arr, n) {
  return Array.from({ length: Math.ceil(arr.length / n) }, (_, i) =>
    arr.slice(i * n, i * n + n),
  );
}
chunk([1, 2, 3, 4, 5], 2); // [[1,2],[3,4],[5]]

// Flatten once (equivalent to flat(1))
[].concat(
  ...[
    [1, 2],
    [3, 4],
  ],
); // [1, 2, 3, 4]

// Zip two arrays together
const keys = ["a", "b", "c"];
const values = [1, 2, 3];
const zipped = keys.map((k, i) => [k, values[i]]); // [['a',1],['b',2],['c',3]]
Object.fromEntries(zipped); // { a: 1, b: 2, c: 3 }

// Partition array into two groups
function partition(arr, predicate) {
  return arr.reduce(
    ([pass, fail], item) =>
      predicate(item) ? [[...pass, item], fail] : [pass, [...fail, item]],
    [[], []],
  );
}
const [evens, odds] = partition([1, 2, 3, 4, 5], (n) => n % 2 === 0);
// evens = [2,4], odds = [1,3,5]

// Intersection (elements in both arrays)
const intersection = arr1.filter((item) => arr2.includes(item));

// Difference (elements in arr1 not in arr2)
const difference = arr1.filter((item) => !arr2.includes(item));
```

---

## 10. Common Mistakes

### Mistake 1 — sort() default sorts as strings

```javascript
// ❌ Wrong numeric sort
[100, 20, 3].sort(); // [100, 20, 3] — "100" < "20" < "3" as strings!
// ✅ Always pass a comparator for numbers
[100, 20, 3].sort((a, b) => a - b); // [3, 20, 100]
```

### Mistake 2 — map/filter/reduce mutate the original

They don't — but if you mutate objects INSIDE them, you mutate the original:

```javascript
const users = [{ name: "Alice", active: false }];

// ❌ Mutating the objects inside map
const updated = users.map((u) => {
  u.active = true;
  return u;
});
// users[0].active is NOW true — original mutated!

// ✅ Return a NEW object
const updated = users.map((u) => ({ ...u, active: true }));
```

### Mistake 3 — reduce without initial value on empty array

```javascript
// ❌ TypeError if array is empty and no initial value
[].reduce((acc, n) => acc + n); // TypeError: Reduce of empty array with no initial value

// ✅ Always provide an initial value
[].reduce((acc, n) => acc + n, 0); // 0
```

---

## 11. Exercises

### Exercise 1 — Transform and aggregate

```javascript
// Given this array, compute the total price of in-stock items over $10
const products = [
  { name: "Widget", price: 5.99, inStock: true },
  { name: "Gadget", price: 19.99, inStock: false },
  { name: "Doohickey", price: 14.99, inStock: true },
  { name: "Thingamajig", price: 8.99, inStock: true },
];
// Expected: 14.99
```

<details>
<summary>Solution</summary>

```javascript
const total = products
  .filter((p) => p.inStock && p.price > 10)
  .reduce((sum, p) => sum + p.price, 0);
// 14.99
```

</details>

---

## 🔗 Related Topics

- [`21-objects-and-destructuring.md`](./21-objects-and-destructuring.md) — Object manipulation patterns
- [`19-functions-fundamentals.md`](./19-functions-fundamentals.md) — Callbacks used in array methods
- [`26-iterators-and-generators.md`](./26-iterators-and-generators.md) — How for...of actually works
