# JavaScript Core — Section Index

> **Complete JavaScript curriculum from beginner fundamentals to senior-level internals. Files are numbered in learning order — start from 01 if you're building from scratch, or jump to your level using the table below.**

---

## 📊 Level Map

| Level               | Files                      | Topics                                                                       |
| ------------------- | -------------------------- | ---------------------------------------------------------------------------- |
| 🟢 **Beginner**     | 01–13                      | Variables, operators, control flow, functions, arrays, objects, strings      |
| 🟡 **Intermediate** | 14, 18, 19, 20, 23, 08–10  | Scope, closures, prototypes, async, error handling, ES6+, modules            |
| 🟠 **Advanced**     | 15, 16, 17, 21, 22, 24, 11 | Call stack, event loop, microtasks, memory, GC, promise internals, iterators |
| 🔴 **Senior**       | 25, 26, 27, 28, 12, 13     | Workers, service workers, design patterns, Proxy/Reflect, typed arrays       |

---

## 🟢 Beginner

| File                                                                   | Topic             | What You'll Learn                                                                      |
| ---------------------------------------------------------------------- | ----------------- | -------------------------------------------------------------------------------------- |
| [`01-variables-and-data-types.md`](./16-variables-and-data-types.md)   | Variables & Types | `var`/`let`/`const`, primitives vs reference, type coercion, `typeof`                  |
| [`02operators-and-expressions.md`](./17-operators-and-expressions.md)  | Operators         | Arithmetic, comparison, logical, ternary, nullish coalescing, optional chaining        |
| [`03-control-flow.md`](./18-control-flow.md)                           | Control Flow      | `if`/`else`, `switch`, `for`/`while`/`do-while`, `break`/`continue`                    |
| [`04-functions-fundamentals.md`](./19-functions-fundamentals.md)       | Functions         | Declarations vs expressions, arrow functions, default params, rest/spread, IIFE        |
| [`05-arrays-and-iteration.md`](./20-arrays-and-iteration.md)           | Arrays            | `map`/`filter`/`reduce`, spread, destructuring, `for...of`, common patterns            |
| [`06-objects-and-destructuring.md`](./21-objects-and-destructuring.md) | Objects           | Creation patterns, shorthand, computed keys, destructuring, spread, `Object.*` methods |
| [`07-strings-and-regex.md`](./22-strings-and-regex.md)                 | Strings & Regex   | Template literals, string methods, regex syntax, patterns, named groups                |

---

## 🟡 Intermediate

| File                                                         | Topic             | What You'll Learn                                                             |
| ------------------------------------------------------------ | ----------------- | ----------------------------------------------------------------------------- |
| [`20-scope-chain.md`](./07-scope-chain.md)                   | Scope             | Lexical scope, block scope, TDZ, scope chain lookup                           |
| [`18-closures.md`](./05-closures.md)                         | Closures          | Closure mechanics, private state, module pattern, common pitfalls             |
| [`14-execution-context.md`](./01-execution-context.md)       | Execution Context | GEC/FEC, variable environment, hoisting explained                             |
| [`19-prototypes.md`](./06-prototypes.md)                     | Prototypes        | `[[Prototype]]`, inheritance chain, `Object.create`, class syntax             |
| [`23-async-patterns.md`](./10-async-patterns.md)             | Async Patterns    | Callbacks → Promises → async/await, error handling, patterns                  |
| [`08-error-handling.md`](./23-error-handling.md)             | Error Handling    | `try`/`catch`/`finally`, error types, custom errors, async errors             |
| [`09-es6-modern-syntax.md`](./24-es6-modern-syntax.md)       | ES6+ Syntax       | Destructuring, spread, template literals, optional chaining, nullish, symbols |
| [`10-modules-and-bundling.md`](./25-modules-and-bundling.md) | Modules           | ESM vs CJS, `import`/`export`, dynamic imports, tree shaking                  |

---

## 🟠 Advanced

| File                                                                 | Topic                  | What You'll Learn                                                          |
| -------------------------------------------------------------------- | ---------------------- | -------------------------------------------------------------------------- |
| [`15-call-stack.md`](./02-call-stack.md)                             | Call Stack             | Stack frames, stack overflow, tail call optimization                       |
| [`16-event-loop.md`](./03-event-loop.md)                             | Event Loop             | Task queue, blocking the main thread, rAF                                  |
| [`17-microtask-vs-macrotask.md`](./04-microtask-vs-macrotask.md)     | Micro vs Macro Tasks   | Precise ordering, queueMicrotask, scheduling                               |
| [`21-memory-management.md`](./08-memory-management.md)               | Memory                 | Stack vs heap, reference counting, mark-and-sweep                          |
| [`22-garbage-collection.md`](./09-garbage-collection.md)             | Garbage Collection     | GC algorithms, generational GC, WeakRef, finalization registry             |
| [`24-promise-internals.md`](./11-promise-internals.md)               | Promise Internals      | States, microtask queue, chaining mechanics, Promise.all                   |
| [`11-iterators-and-generators.md`](./26-iterators-and-generators.md) | Iterators & Generators | Iterator protocol, `Symbol.iterator`, generator functions, lazy evaluation |

---

## 🔴 Senior

| File                                                                                   | Topic            | What You'll Learn                                            |
| -------------------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------ |
| [`25-web-workers.md`](./12-web-workers.md)                                             | Web Workers      | Off-main-thread execution, MessageChannel, SharedArrayBuffer |
| [`26-service-workers.md`](./13-service-workers.md)                                     | Service Workers  | Lifecycle, caching strategies, background sync, push         |
| [`27-observer-patterns.md`](./14-observer-patterns.md)                                 | Observer Pattern | MutationObserver, IntersectionObserver, ResizeObserver       |
| [`28-pub-sub-systems.md`](./15-pub-sub-systems.md)                                     | Pub/Sub Systems  | Event bus architecture, decoupled communication patterns     |
| [`12-proxy-reflect-and-metaprogramming.md`](./27-proxy-reflect-and-metaprogramming.md) | Proxy & Reflect  | Traps, reactive objects, validation, metaprogramming         |
| [`13-typed-arrays-and-binary-data.md`](./28-typed-arrays-and-binary-data.md)           | Typed Arrays     | ArrayBuffer, DataView, TypedArrays, binary protocols         |

---

## 🎯 Suggested Learning Paths

### Complete beginner (no prior JS)

`01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 20 → 18 → 09 → 10 → 23 → 19 → 14 → 15 → 16 → 17 → 24`

### Intermediate upgrading skills

`20 → 18 → 14 → 08 → 09 → 10 → 19 → 23 → 24 → 21 → 16 → 17 → 11`

### Senior / interview prep

`16 → 17 → 21 → 22 → 24 → 11 → 12 → 13 → 25 → 26 → 27 → 28`
