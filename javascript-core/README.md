# JavaScript Core — Section Index

> **Complete JavaScript curriculum from beginner fundamentals to senior-level internals. Files are numbered in learning order — start from 01 if you're building from scratch, or jump to your level using the table below.**

---

## 📊 Level Map

| Level               | Files                      | Topics                                                                       |
| ------------------- | -------------------------- | ---------------------------------------------------------------------------- |
| 🟢 **Beginner**     | 01–11                      | Variables, operators, control flow, functions, arrays, objects, strings      |
| 🟡 **Intermediate** | 12, 16, 17, 18, 21, 08–10  | Scope, closures, prototypes, async, error handling, ES6+, modules            |
| 🟠 **Advanced**     | 13, 14, 15, 19, 20, 22, 11 | Call stack, event loop, microtasks, memory, GC, promise internals, iterators |
| 🔴 **Senior**       | 23, 24, 25, 26, 27, 28     | Workers, service workers, design patterns, Proxy/Reflect, typed arrays       |

---

## 🟢 Beginner

| File                                                                   | Topic             | What You'll Learn                                                                      |
| ---------------------------------------------------------------------- | ----------------- | -------------------------------------------------------------------------------------- |
| [`01-variables-and-data-types.md`](./01-variables-and-data-types.md)   | Variables & Types | `var`/`let`/`const`, primitives vs reference, type coercion, `typeof`                  |
| [`02-operators-and-expressions.md`](./02-operators-and-expressions.md) | Operators         | Arithmetic, comparison, logical, ternary, nullish coalescing, optional chaining        |
| [`03-control-flow.md`](./03-control-flow.md)                           | Control Flow      | `if`/`else`, `switch`, `for`/`while`/`do-while`, `break`/`continue`                    |
| [`04-functions-fundamentals.md`](./04-functions-fundamentals.md)       | Functions         | Declarations vs expressions, arrow functions, default params, rest/spread, IIFE        |
| [`05-arrays-and-iteration.md`](./05-arrays-and-iteration.md)           | Arrays            | `map`/`filter`/`reduce`, spread, destructuring, `for...of`, common patterns            |
| [`06-objects-and-destructuring.md`](./06-objects-and-destructuring.md) | Objects           | Creation patterns, shorthand, computed keys, destructuring, spread, `Object.*` methods |
| [`07-strings-and-regex.md`](./07-strings-and-regex.md)                 | Strings & Regex   | Template literals, string methods, regex syntax, patterns, named groups                |

---

## 🟡 Intermediate

| File                                                         | Topic             | What You'll Learn                                                             |
| ------------------------------------------------------------ | ----------------- | ----------------------------------------------------------------------------- |
| [`18-scope-chain.md`](./18-scope-chain.md)                   | Scope             | Lexical scope, block scope, TDZ, scope chain lookup                           |
| [`16-closures.md`](./16-closures.md)                         | Closures          | Closure mechanics, private state, module pattern, common pitfalls             |
| [`12-execution-context.md`](./12-execution-context.md)       | Execution Context | GEC/FEC, variable environment, hoisting explained                             |
| [`17-prototypes.md`](./17-prototypes.md)                     | Prototypes        | `[[Prototype]]`, inheritance chain, `Object.create`, class syntax             |
| [`21-async-patterns.md`](./21-async-patterns.md)             | Async Patterns    | Callbacks → Promises → async/await, error handling, patterns                  |
| [`08-error-handling.md`](./08-error-handling.md)             | Error Handling    | `try`/`catch`/`finally`, error types, custom errors, async errors             |
| [`09-es6-modern-syntax.md`](./09-es6-modern-syntax.md)       | ES6+ Syntax       | Destructuring, spread, template literals, optional chaining, nullish, symbols |
| [`10-modules-and-bundling.md`](./10-modules-and-bundling.md) | Modules           | ESM vs CJS, `import`/`export`, dynamic imports, tree shaking                  |

---

## 🟠 Advanced

| File                                                                 | Topic                  | What You'll Learn                                                          |
| -------------------------------------------------------------------- | ---------------------- | -------------------------------------------------------------------------- |
| [`13-call-stack.md`](./13-call-stack.md)                             | Call Stack             | Stack frames, stack overflow, tail call optimization                       |
| [`14-event-loop.md`](./14-event-loop.md)                             | Event Loop             | Task queue, blocking the main thread, rAF                                  |
| [`15-microtask-vs-macrotask.md`](./15-microtask-vs-macrotask.md)     | Micro vs Macro Tasks   | Precise ordering, queueMicrotask, scheduling                               |
| [`19-memory-management.md`](./19-memory-management.md)               | Memory                 | Stack vs heap, reference counting, mark-and-sweep                          |
| [`20-garbage-collection.md`](./20-garbage-collection.md)             | Garbage Collection     | GC algorithms, generational GC, WeakRef, finalization registry             |
| [`22-promise-internals.md`](./22-promise-internals.md)               | Promise Internals      | States, microtask queue, chaining mechanics, Promise.all                   |
| [`11-iterators-and-generators.md`](./11-iterators-and-generators.md) | Iterators & Generators | Iterator protocol, `Symbol.iterator`, generator functions, lazy evaluation |

---

## 🔴 Senior

| File                                                                                   | Topic            | What You'll Learn                                            |
| -------------------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------ |
| [`23-web-workers.md`](./23-web-workers.md)                                             | Web Workers      | Off-main-thread execution, MessageChannel, SharedArrayBuffer |
| [`24-service-workers.md`](./24-service-workers.md)                                     | Service Workers  | Lifecycle, caching strategies, background sync, push         |
| [`25-observer-patterns.md`](./25-observer-patterns.md)                                 | Observer Pattern | MutationObserver, IntersectionObserver, ResizeObserver       |
| [`26-pub-sub-systems.md`](./26-pub-sub-systems.md)                                     | Pub/Sub Systems  | Event bus architecture, decoupled communication patterns     |
| [`27-proxy-reflect-and-metaprogramming.md`](./27-proxy-reflect-and-metaprogramming.md) | Proxy & Reflect  | Traps, reactive objects, validation, metaprogramming         |
| [`28-typed-arrays-and-binary-data.md`](./28-typed-arrays-and-binary-data.md)           | Typed Arrays     | ArrayBuffer, DataView, TypedArrays, binary protocols         |

---

## 🎯 Suggested Learning Paths

### Complete beginner (no prior JS)

`01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 18 → 16 → 09 → 10 → 21 → 17 → 12 → 13 → 14 → 15 → 22`

### Intermediate upgrading skills

`18 → 16 → 12 → 08 → 09 → 10 → 17 → 21 → 22 → 19 → 14 → 15 → 11`

### Senior / interview prep

`14 → 15 → 19 → 20 → 22 → 11 → 27 → 28 → 23 → 24 → 25 → 26`
