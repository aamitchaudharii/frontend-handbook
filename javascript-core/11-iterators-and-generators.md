# 26 — Iterators and Generators

> **"An iterator is a promise: 'I'll give you values one at a time, and I'll tell you when I'm done.' A generator is a function that makes that promise easy to keep — it suspends itself at each `yield`, handing back a value and resuming exactly where it paused. This is lazy evaluation: nothing is computed until you ask for it."**

🟠 **Level: Advanced**

---

## 📚 Table of Contents

1. [The Iterator Protocol](#1-the-iterator-protocol)
2. [The Iterable Protocol](#2-the-iterable-protocol)
3. [Built-in Iterables](#3-built-in-iterables)
4. [Generator Functions](#4-generator-functions)
5. [Two-Way Communication with Generators](#5-two-way-communication-with-generators)
6. [Generator Return and Throw](#6-generator-return-and-throw)
7. [Infinite Sequences](#7-infinite-sequences)
8. [Async Generators](#8-async-generators)
9. [Practical Use Cases](#9-practical-use-cases)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. The Iterator Protocol

An **iterator** is any object with a `next()` method that returns `{ value, done }`:

```javascript
// A manual iterator
function rangeIterator(start, end) {
  let current = start;
  return {
    next() {
      if (current <= end) {
        return { value: current++, done: false };
      }
      return { value: undefined, done: true };
    },
  };
}

const iter = rangeIterator(1, 3);
iter.next(); // { value: 1, done: false }
iter.next(); // { value: 2, done: false }
iter.next(); // { value: 3, done: false }
iter.next(); // { value: undefined, done: true }

// After done: true, subsequent calls should keep returning { value: undefined, done: true }

// OPTIONAL: iterators can also be iterable by returning themselves from [Symbol.iterator]
function rangeIterator(start, end) {
  let current = start;
  return {
    next() {
      return current <= end
        ? { value: current++, done: false }
        : { value: undefined, done: true };
    },
    [Symbol.iterator]() {
      return this;
    }, // makes it usable in for...of
  };
}
```

---

## 2. The Iterable Protocol

An **iterable** is any object with a `[Symbol.iterator]()` method that returns an iterator:

```javascript
// Custom iterable class
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  // [Symbol.iterator] must return a fresh iterator on each call
  // (so the iterable can be iterated multiple times)
  [Symbol.iterator]() {
    let current = this.start;
    const { end, step } = this;

    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      },
      [Symbol.iterator]() {
        return this;
      },
    };
  }
}

const range = new Range(0, 10, 2);

// Works with all iteration consumers:
for (const n of range) {
  console.log(n);
} // 0, 2, 4, 6, 8, 10
[...range]; // [0, 2, 4, 6, 8, 10]
const [a, b] = range; // a = 0, b = 2 (destructuring)
Array.from(range); // [0, 2, 4, 6, 8, 10]
new Set(range); // Set {0, 2, 4, 6, 8, 10}
Math.max(...range); // 10

// Calling [Symbol.iterator]() twice = two independent iterators
const iter1 = range[Symbol.iterator]();
const iter2 = range[Symbol.iterator]();
iter1.next(); // { value: 0, done: false }
iter2.next(); // { value: 0, done: false } — independent state
iter1.next(); // { value: 2, done: false }
```

---

## 3. Built-in Iterables

```javascript
// All of these implement [Symbol.iterator]:
Array, String, Map, Set, TypedArray, arguments, NodeList, HTMLCollection

// Array: iterates values
[...['a', 'b', 'c']];         // ['a', 'b', 'c']

// String: iterates Unicode code points (handles emoji correctly!)
[...'hello😀'];                 // ['h','e','l','l','o','😀']
// (contrast with: 'hello😀'.split('') which breaks emoji into surrogates)

// Map: iterates [key, value] pairs
[...new Map([['a',1],['b',2]])]; // [['a',1],['b',2]]
// also: map.keys(), map.values(), map.entries() all return iterables

// Set: iterates values in insertion order
[...new Set([3, 1, 2, 1, 3])]; // [3, 1, 2]

// Generator (see next section) is also an iterable

// Checking if something is iterable:
function isIterable(obj) {
  return obj != null && typeof obj[Symbol.iterator] === 'function';
}
isIterable([1, 2, 3]); // true
isIterable('hello');   // true
isIterable(42);        // false
```

---

## 4. Generator Functions

Generators are functions that can **pause** execution with `yield` and **resume** later:

```javascript
// Syntax: function* (or function *name, or as a method)
function* simple() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = simple(); // calling a generator returns a generator OBJECT
// The function body does NOT run yet!

gen.next(); // { value: 1, done: false } — runs until first yield
gen.next(); // { value: 2, done: false } — resumes from after first yield
gen.next(); // { value: 3, done: false }
gen.next(); // { value: undefined, done: true } — function completed

// Generator objects are BOTH iterators AND iterables
for (const n of simple()) {
  console.log(n);
} // 1, 2, 3
[...simple()]; // [1, 2, 3]

// yield* — delegate to another iterable
function* concat(...iterables) {
  for (const it of iterables) {
    yield* it; // delegates: yields each value from `it`
  }
}
[...concat([1, 2], [3, 4], [5])]; // [1, 2, 3, 4, 5]

// yield* also returns the value the delegated generator RETURNS (not yields)
function* inner() {
  yield 1;
  return "inner done"; // this is the value yield* receives
}
function* outer() {
  const result = yield* inner();
  console.log(result); // 'inner done'
  yield 2;
}
[...outer()]; // [1, 2] — 'inner done' is not yielded, just returned
```

---

## 5. Two-Way Communication with Generators

`next(value)` can SEND a value INTO the generator — it becomes the result of the `yield` expression:

```javascript
function* calculator() {
  let result = 0;
  while (true) {
    const input = yield result; // suspend, return result, receive next input
    if (input === null) break; // null signals termination
    result += input;
  }
  return result;
}

const calc = calculator();
calc.next(); // { value: 0, done: false }  — start the generator
calc.next(10); // { value: 10, done: false } — input=10, result becomes 10
calc.next(5); // { value: 15, done: false } — input=5, result becomes 15
calc.next(3); // { value: 18, done: false }
calc.next(null); // { value: 18, done: true }  — terminates, returns 18

// The first next() call starts the generator and runs until the first yield.
// The value passed to the FIRST next() is always ignored
// (there's no `yield` expression waiting for it yet).
```

---

## 6. Generator Return and Throw

```javascript
function* gen() {
  try {
    yield 1;
    yield 2; // never reached if return() or throw() is called first
    yield 3;
  } finally {
    console.log("cleanup"); // finally ALWAYS runs
  }
}

const g = gen();
g.next(); // { value: 1, done: false }
g.return(99); // 'cleanup' logged; { value: 99, done: true }
// return() forces the generator to finish with the given value
// finally still runs

const g2 = gen();
g2.next(); // { value: 1, done: false }
try {
  g2.throw(new Error("oops")); // throws the error AT the suspended yield point
  // 'cleanup' logged
} catch (e) {
  console.log(e.message); // 'oops' — if gen didn't catch it, propagates here
}
```

---

## 7. Infinite Sequences

One of the most powerful generator use cases — lazily evaluated, never-ending sequences:

```javascript
// Infinite counter
function* count(start = 0, step = 1) {
  while (true) yield (start += step);
}

// Take first n values from an iterable
function* take(n, iterable) {
  let count = 0;
  for (const item of iterable) {
    if (count++ >= n) return;
    yield item;
  }
}

// Compose lazily
[...take(5, count(0, 2))]; // [2, 4, 6, 8, 10]

// Fibonacci sequence (infinite)
function* fibonacci() {
  let [prev, curr] = [0, 1];
  while (true) {
    yield curr;
    [prev, curr] = [curr, prev + curr];
  }
}
[...take(8, fibonacci())]; // [1, 1, 2, 3, 5, 8, 13, 21]

// Infinite generator with filtering
function* filter(predicate, iterable) {
  for (const item of iterable) {
    if (predicate(item)) yield item;
  }
}
function* map(fn, iterable) {
  for (const item of iterable) {
    yield fn(item);
  }
}

// First 5 even perfect squares: 4, 16, 36, 64, 100
const pipeline = take(
  5,
  filter(
    (n) => n % 2 === 0,
    map((n) => n * n, count(1)),
  ),
);
[...pipeline]; // [4, 16, 36, 64, 100]
```

---

## 8. Async Generators

```javascript
// async function* — yields Promises, used with for await...of
async function* paginate(url) {
  let page = 1;
  while (true) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();

    if (!data.items.length) return; // no more data

    yield data.items; // yields one page at a time
    page++;
  }
}

// Consume with for await...of
for await (const items of paginate('/api/users')) {
  processItems(items);
}

// Streaming data processing (e.g., reading a large file line by line)
async function* readLines(stream) {
  const reader = stream.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) {
      if (buffer) yield buffer; // yield remaining data
      return;
    }
    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop()!; // save incomplete line for next chunk
    for (const line of lines) {
      if (line) yield line;
    }
  }
}

for await (const line of readLines(response.body)) {
  parseLine(line);
}
```

---

## 9. Practical Use Cases

```javascript
// USE CASE 1: Unique ID generator
function* idGenerator(prefix = "") {
  let id = 0;
  while (true) {
    yield `${prefix}${++id}`;
  }
}
const userId = idGenerator("user-");
userId.next().value; // 'user-1'
userId.next().value; // 'user-2'

// USE CASE 2: State machine
function* trafficLight() {
  while (true) {
    yield "green";
    yield "yellow";
    yield "red";
  }
}
const light = trafficLight();
light.next().value; // 'green'
light.next().value; // 'yellow'
light.next().value; // 'red'
light.next().value; // 'green' (loops)

// USE CASE 3: Middleware-style pipeline
function* middleware(value) {
  value = yield value * 2; // step 1: double
  value = yield value + 10; // step 2: add 10
  return value;
}

// USE CASE 4: Cooperative multitasking (basis of early async/await polyfills)
function run(generator) {
  const iter = generator();
  function step(result) {
    const { value, done } = iter.next(result);
    if (!done) {
      Promise.resolve(value).then(step); // when promise resolves, resume generator
    }
  }
  step();
}
// This is conceptually how Babel transpiles async/await!

// USE CASE 5: Lazy range that works like Python's range()
function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) yield i;
}
[...range(0, 10, 2)]; // [0, 2, 4, 6, 8]
```

---

## 10. Common Mistakes

### Mistake 1 — Reusing an exhausted generator

```javascript
function* gen() {
  yield 1;
  yield 2;
}
const g = gen();
[...g]; // [1, 2]
[...g]; // [] — generator is EXHAUSTED, done: true forever

// ✅ Call gen() again for a new generator instance
[...gen()]; // [1, 2]
[...gen()]; // [1, 2]
```

### Mistake 2 — Forgetting the first `next()` starts execution

```javascript
// The generator body doesn't run until the first next() call
function* gen() {
  console.log("start");
  yield 1;
}
const g = gen(); // nothing logged yet
g.next(); // 'start' logged, { value: 1, done: false }
```

### Mistake 3 — Return value vs yield

```javascript
function* gen() {
  yield 1;
  yield 2;
  return "done"; // the return value
}
[...gen()]; // [1, 2] — 'done' is NOT in the spread result!
// Return value appears as done: true in the iterator protocol
// but is NOT included in spread, Array.from, or for...of
const g = gen();
g.next(); // { value: 1, done: false }
g.next(); // { value: 2, done: false }
g.next(); // { value: 'done', done: true } — only visible via .next()
```

---

## 11. Exercises

### Exercise 1 — Lazy flat map

```javascript
// Implement a generator function lazyFlatMap(iterable, fn) that is
// equivalent to Array.prototype.flatMap but works lazily on any iterable.
// It should work with infinite iterables (only compute what's consumed).

// Example: take the first 5 results of expanding each number into [n, n*n]
// from an infinite counter starting at 1.
// Expected first 5: [1, 1, 2, 4, 3]
```

<details>
<summary>Solution</summary>

```javascript
function* lazyFlatMap(iterable, fn) {
  for (const item of iterable) {
    yield* fn(item); // delegate to the iterable returned by fn
  }
}

function* count(start = 1) {
  while (true) yield start++;
}
function* take(n, it) {
  let i = 0;
  for (const x of it) {
    if (i++ >= n) return;
    yield x;
  }
}

const expanded = lazyFlatMap(count(), (n) => [n, n * n]);
[...take(10, expanded)]; // [1, 1, 2, 4, 3, 9, 4, 16, 5, 25]
// Works with infinite source — never tries to create the full array
```

</details>

---

## 🔗 Related Topics

- [`24-es6-modern-syntax.md`](./24-es6-modern-syntax.md) — `for...of` and Symbols
- [`11-promise-internals.md`](./11-promise-internals.md) — How async/await relates to generators
- [`28-typed-arrays-and-binary-data.md`](./28-typed-arrays-and-binary-data.md) — TypedArrays are iterables
