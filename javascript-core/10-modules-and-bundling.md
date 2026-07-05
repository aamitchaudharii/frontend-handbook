# 25 — Modules and Bundling

> **"Modules are the unit of encapsulation in JavaScript. Everything you import is a deliberate contract. Everything you don't export is private. Understanding how modules actually work — how they load, how they cache, how bundlers transform them — is what lets you reason about bundle size, circular dependencies, and tree-shaking."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [Why Modules?](#1-why-modules)
2. [ES Modules (ESM) — import / export](#2-es-modules-esm--import--export)
3. [CommonJS (CJS) — require / module.exports](#3-commonjs-cjs--require--moduleexports)
4. [ESM vs CJS — Key Differences](#4-esm-vs-cjs--key-differences)
5. [Dynamic Imports](#5-dynamic-imports)
6. [How the Module System Works](#6-how-the-module-system-works)
7. [Circular Dependencies](#7-circular-dependencies)
8. [Tree Shaking](#8-tree-shaking)
9. [Bundlers (Vite, Webpack, Rollup, esbuild)](#9-bundlers-vite-webpack-rollup-esbuild)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. Why Modules?

```javascript
// Before modules: everything lived on the global scope
// script1.js:
var helper = function (x) {
  return x * 2;
};

// script2.js:
var helper = "overwritten!"; // silently overwrites script1.js's helper

// With modules: each file has its own scope
// helper.js:
export function helper(x) {
  return x * 2;
} // scoped to this file

// main.js:
import { helper } from "./helper.js"; // explicit dependency declaration
```

---

## 2. ES Modules (ESM) — import / export

```javascript
// ===== NAMED EXPORTS =====
// math.js
export const PI = 3.14159;
export function add(a, b)      { return a + b; }
export function subtract(a, b) { return a - b; }
export class Vector { constructor(x, y) { this.x = x; this.y = y; } }

// Group export (define first, export at the end):
function multiply(a, b) { return a * b; }
function divide(a, b)   { return a / b; }
export { multiply, divide };

// Export with rename:
export { multiply as times, divide as over };

// ===== DEFAULT EXPORT =====
// One default export per module (the "main thing" the module provides)
// user.js
export default class User {
  constructor(name) { this.name = name; }
}
// OR: export default function createUser(name) { return { name }; }
// OR: const config = { ... }; export default config;

// ===== NAMED IMPORTS =====
import { PI, add, subtract } from './math.js';
import { multiply as times } from './math.js'; // rename on import

// Import everything as a namespace
import * as Math from './math.js';
Math.add(1, 2); // 3
Math.PI;        // 3.14159

// ===== DEFAULT IMPORT =====
import User from './user.js'; // any name — it's the default
import MyUser from './user.js'; // same thing, different local name

// ===== COMBINED =====
// A module can have both named and default exports
// utils.js:
export const VERSION = '1.0.0';
export default function main() { /* ... */ }

import main, { VERSION } from './utils.js';

// ===== RE-EXPORTING (barrel files) =====
// index.js that re-exports from sub-files:
export { add, subtract } from './math.js';
export { default as User } from './user.js';
export * from './helpers.js';
// Consumers import from the barrel: import { add, User } from './lib';
```

---

## 3. CommonJS (CJS) — require / module.exports

```javascript
// Still dominant in Node.js codebases

// ===== EXPORTING =====
// math.js
const PI = 3.14159;
function add(a, b) {
  return a + b;
}

module.exports = { PI, add }; // export an object
// OR: module.exports = add;              // export a single value
// OR: exports.PI = PI; exports.add = add; // shorthand (same thing as first)

// ===== IMPORTING =====
const { PI, add } = require("./math"); // destructure the exports object
const math = require("./math"); // import the whole object
const add = require("./math").add; // inline property access

// Dynamic require (CJS is synchronous — require can be called anywhere):
if (process.env.NODE_ENV === "test") {
  const mock = require("./mock");
}

// require() is synchronous — blocks execution until the module loads
// NOT suitable for browsers (files must be downloaded asynchronously)
// This is why bundlers exist: they process CJS/ESM at build time
```

---

## 4. ESM vs CJS — Key Differences

```
                ESM (import/export)     CJS (require/module.exports)
─────────────────────────────────────────────────────────────────────
Syntax:         import / export         require() / module.exports
Loading:        Asynchronous            Synchronous
Analysis:       Static (compile time)   Dynamic (runtime)
Scope:          Block-scoped imports    Function scope (require is a fn)
Live bindings:  YES — imported names    NO — a snapshot copy
                reflect current value
Tree-shakeable: YES                     NO (most bundlers can't)
Top-level await: YES (.mjs or type:module) NO
Browser support: YES (native)           NO (needs bundler)
Node.js:        YES (.mjs or package.json type:"module") YES (.js, .cjs)
─────────────────────────────────────────────────────────────────────

LIVE BINDINGS (ESM-only, important):
// counter.js
export let count = 0;
export function increment() { count++; }

// main.js
import { count, increment } from './counter.js';
console.log(count); // 0
increment();
console.log(count); // 1 ← reflects the CURRENT value (live binding)

// In CJS: you'd get a snapshot copy of count = 0, increment() wouldn't
// update the local variable
```

---

## 5. Dynamic Imports

```javascript
// Static import: resolved at parse time, always at the top level
import { heavy } from "./heavy.js"; // loaded immediately

// Dynamic import: returns a Promise, can be used anywhere, loaded on demand
// (Code splitting — the key to lazy loading)
const { heavy } = await import("./heavy.js");

// React lazy loading (code splitting by route):
const Dashboard = React.lazy(() => import("./pages/Dashboard"));
const Settings = React.lazy(() => import("./pages/Settings"));

// Conditional imports:
const plugin = userPrefs.darkMode
  ? await import("./themes/dark.js")
  : await import("./themes/light.js");

// Import based on user action (reduce initial bundle size):
button.addEventListener("click", async () => {
  const { renderChart } = await import("./chart-library.js");
  renderChart(data); // library only loaded when user needs it
});

// Dynamic import always returns the module's namespace object:
const module = await import("./math.js");
module.default; // default export
module.add; // named export

// With error handling:
try {
  const { util } = await import("./optional-util.js");
  util.run();
} catch {
  // Module failed to load — use fallback
}
```

---

## 6. How the Module System Works

```
ESM LOADING PHASES (browser):

1. CONSTRUCTION (parse)
   Browser fetches and parses each module file.
   Builds a module graph (dependency tree) without executing any code.
   ES modules are STATIC: all imports/exports must be at the top level,
   so the full dependency graph is known at parse time.

2. INSTANTIATION (link)
   Each module's exported names are wired to memory locations.
   Live binding: importing `count` from counter.js doesn't copy the value —
   it creates a REFERENCE to the same memory location counter.js uses.

3. EVALUATION (execute)
   Each module's code runs ONCE, in depth-first order.
   Results are cached: importing the same module twice returns the
   SAME cached module instance.

MODULE CACHE:
  Each module URL is resolved once and cached.
  import './util.js' from different files gets the SAME instance.
  Modules are singletons per URL.

RESOLUTION ORDER:
  import './relative'  → relative to current file
  import 'lodash'      → node_modules lookup (in Node.js)
                         bundler alias/resolve in browser contexts
```

---

## 7. Circular Dependencies

```javascript
// Circular: A imports B, B imports A

// a.js
import { bValue } from "./b.js";
export const aValue = "from A";
console.log("A uses:", bValue); // may be undefined if B hasn't evaluated yet!

// b.js
import { aValue } from "./a.js";
export const bValue = "from B";
console.log("B uses:", aValue); // may be undefined

// ESM HANDLES CIRCULARS VIA LIVE BINDINGS:
// Both modules are partially evaluated. The second-evaluated module
// MAY see undefined for imports from the first module at the time
// of evaluation, but will see the correct live value once evaluation completes.

// ✅ Avoid circulars by:
// 1. Extracting shared logic to a third file that both import from
// 2. Using dependency injection instead of direct imports
// 3. Restructuring so dependencies only flow in one direction

// Detecting circulars: bundlers like webpack/rollup warn about them.
// In Node.js: circular require() with CJS gives partial objects.
```

---

## 8. Tree Shaking

```javascript
// Tree shaking: bundlers remove unused exports from the final bundle

// ✅ Named exports are tree-shakeable
// utils.js
export function formatDate(d) { /* ... */ }
export function formatCurrency(n) { /* ... */ }
export function formatPhone(p) { /* ... */ }

// Only formatDate is imported → formatCurrency and formatPhone are excluded
import { formatDate } from './utils.js';

// ❌ Default export of an object is NOT tree-shakeable
export default {
  formatDate(d) { /* ... */ },
  formatCurrency(n) { /* ... */ },
};
// Bundler can't know which properties are used at compile time

// Requirements for tree shaking to work:
// 1. ES modules (NOT CommonJS — CJS can't be statically analyzed)
// 2. "sideEffects": false in package.json (tells bundler the package is safe to shake)
// 3. Production build mode (dev builds skip tree shaking for speed)
// 4. No side effects in module scope (code that runs on import, not in functions)

// Side effect example (prevents tree shaking of the module):
// styles.css imported for its side effect (injecting styles):
import './styles.css'; // this IS a side effect — always included

// package.json declaring which files have side effects:
{
  "sideEffects": ["*.css", "*.scss"],  // only CSS files have side effects
  // implicitly: all JS files are side-effect-free → tree-shakeable
}
```

---

## 9. Bundlers (Vite, Webpack, Rollup, esbuild)

```
WHAT BUNDLERS DO:
  1. Start from an entry point (e.g., index.js)
  2. Follow all imports recursively → build the full dependency graph
  3. Transform code (TypeScript → JS, JSX → JS, new syntax → older syntax)
  4. Bundle into one (or more) output files
  5. Optimize: minify, tree-shake, split code into chunks

TOOL COMPARISON:
  Vite (dev) + Rollup (prod):
    Dev: native ESM in browser — no bundling during development (blazing fast HMR)
    Prod: Rollup bundling with excellent tree shaking
    Best for: new projects, libraries, SPA frontends

  Webpack:
    Most configurable, largest ecosystem
    Slower than Vite but extremely mature
    Best for: large existing projects, complex configurations

  Rollup:
    Designed for libraries (not apps)
    Best tree shaking
    Best for: npm libraries, design systems

  esbuild:
    Written in Go — 10-100x faster than webpack
    Limited configurability
    Used as the core of Vite and many other tools

CODE SPLITTING STRATEGIES:
  By route:        each page/route is a separate chunk (most common)
  By vendor:       node_modules in a separate "vendor" chunk (long-term caching)
  Dynamic import:  each dynamic import() becomes its own chunk
  By user action:  import heavy libraries only when needed
```

---

## 10. Common Mistakes

### Mistake 1 — Barrel file that defeats tree shaking

```javascript
// ❌ index.js imports everything eagerly, even if consumer only needs one
import * as all from "./all-components"; // forces ALL components to load

// ✅ Named re-exports allow tree shaking
export { Button } from "./Button";
export { Input } from "./Input";
export { Modal } from "./Modal";
// Consumer: import { Button } from './components' → only Button's code loaded
```

### Mistake 2 — Side effects in module scope preventing tree shaking

```javascript
// ❌ This code runs when the module is imported — bundler must include it
console.log("Module loaded"); // side effect in module scope
analytics.track("module-import"); // also a side effect

// ✅ Keep module scope clean — only define functions/constants
export function trackLoad() {
  analytics.track("module-import");
}
// Consumers call trackLoad() explicitly when they want the side effect
```

### Mistake 3 — Mixing ESM and CJS in Node.js

```javascript
// ❌ Can't use import inside a .js file if package.json says "type": "commonjs"
import express from "express"; // SyntaxError in a CJS module

// ✅ Options:
// 1. Add "type": "module" to package.json (ESM everywhere)
// 2. Rename file to .mjs for individual ESM files
// 3. Use require() for CJS: const express = require('express')
// 4. Use interop: const { default: express } = await import('express') inside async
```

---

## 11. Exercises

### Exercise 1 — Refactor to tree-shakeable exports

```javascript
// Refactor this module to allow tree shaking of individual utilities
const utils = {
  formatDate(d) {
    return d.toISOString().split("T")[0];
  },
  formatCurrency(n) {
    return new Intl.NumberFormat("en-US", {
      style: "currency",
      currency: "USD",
    }).format(n);
  },
  capitalise(s) {
    return s.charAt(0).toUpperCase() + s.slice(1);
  },
};
export default utils;
```

<details>
<summary>Solution</summary>

```javascript
// Named exports — each can be independently tree-shaken
export function formatDate(d) {
  return d.toISOString().split("T")[0];
}

export function formatCurrency(n) {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  }).format(n);
}

export function capitalise(s) {
  return s.charAt(0).toUpperCase() + s.slice(1);
}

// Consumer imports only what they need:
import { formatDate } from "./utils.js";
// formatCurrency and capitalise are excluded from the bundle
```

</details>

---

## 🔗 Related Topics

- [`12-web-workers.md`](./12-web-workers.md) — Using dynamic import inside workers
- [`24-es6-modern-syntax.md`](./24-es6-modern-syntax.md) — ES6 features modules build on
- [`performance/08-bundle-optimization.md`](../performance/08-bundle-optimization.md) — Bundle size optimization strategies
