# 17 — tsconfig Deep Dive

> **"tsconfig.json is the contract between your code and the TypeScript compiler. Most developers copy a tsconfig from a starter template and never revisit it — which means they're making unconscious tradeoffs about strictness, module resolution, and build performance that they don't understand. Knowing every option that matters means you can explain every error, tune performance, and migrate between environments without hitting mysterious walls."**

🔴 **Level: Senior**

---

## 📚 Table of Contents

1. [tsconfig Structure](#1-tsconfig-structure)
2. [Strict Mode — What It Actually Enables](#2-strict-mode--what-it-actually-enables)
3. [Module Options](#3-module-options)
4. [Output Options](#4-output-options)
5. [Type Checking Options](#5-type-checking-options)
6. [Path Aliases](#6-path-aliases)
7. [Project References](#7-project-references)
8. [Common Configurations by Project Type](#8-common-configurations-by-project-type)
9. [Performance Tuning](#9-performance-tuning)
10. [Migration — Adding TypeScript to an Existing Project](#10-migration--adding-typescript-to-an-existing-project)
11. [Common Mistakes](#11-common-mistakes)

---

## 1. tsconfig Structure

```json
{
  // Compiler behavior
  "compilerOptions": {
    /* ... */
  },

  // Which files to include / exclude
  "include": ["src", "tests"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"],

  // Specific file list (overrides include/exclude)
  "files": ["src/main.ts", "src/types/global.d.ts"],

  // Extend another tsconfig (inherit + override)
  "extends": "./tsconfig.base.json",

  // Project references (for monorepos)
  "references": [{ "path": "../packages/ui" }, { "path": "../packages/utils" }]
}
```

```
RESOLUTION ORDER:
  1. files (if specified, overrides include/exclude)
  2. include + exclude (glob patterns)
  3. Referenced project tsconfigs (for project references)

INHERITANCE (extends):
  A child tsconfig inherits ALL options from the parent.
  Child options OVERRIDE parent options.
  `include`, `exclude`, `files` are NOT inherited — only `compilerOptions`.
```

---

## 2. Strict Mode — What It Actually Enables

```typescript
// "strict": true is a shorthand for ALL of these:
{
  "compilerOptions": {
    "strict": true,
    // Expands to:
    "noImplicitAny": true,              // ①
    "strictNullChecks": true,           // ②
    "strictFunctionTypes": true,        // ③
    "strictBindCallApply": true,        // ④
    "strictPropertyInitialization": true, // ⑤
    "noImplicitThis": true,             // ⑥
    "useUnknownInCatchVariables": true,  // ⑦
    "alwaysStrict": true,               // ⑧
  }
}
```

```typescript
// ① noImplicitAny: parameters without types default to `any` — error!
function process(data) {} // ❌ 'data' implicitly has an 'any' type

// ② strictNullChecks: null/undefined are NOT in other types
let name: string = null; // ❌ null not assignable to string
let name: string | null = null; // ✅ must be explicit

// ③ strictFunctionTypes: parameter types are CONTRAVARIANT (more correct)
type Animal = { name: string };
type Dog = Animal & { breed: string };
type DogFn = (d: Dog) => void;
type AnimalFn = (a: Animal) => void;

let animalFn: AnimalFn = (a) => console.log(a.name);
let dogFn: DogFn = animalFn; // ✅ safe: AnimalFn accepts any Animal, Dog is an Animal
// animalFn = dogFn; // ❌ unsafe: dogFn expects a Dog but animalFn only promises an Animal

// ④ strictBindCallApply: .bind()/.call()/.apply() are type-checked
function add(a: number, b: number) {
  return a + b;
}
add.call(null, 1, "two"); // ❌ 'two' should be a number

// ⑤ strictPropertyInitialization: class properties must be initialized
class User {
  name: string; // ❌ not initialized in constructor
  constructor() {} // must set this.name here or give a default
}

// ⑥ noImplicitThis: `this` in functions must have a declared type
function greet(this: { name: string }) {
  return `Hello, ${this.name}`; // ✅ `this` is typed
}

// ⑦ useUnknownInCatchVariables: catch(e) — `e` is `unknown` not `any`
try {
} catch (e) {
  e.message; // ❌ e is unknown — must narrow first
  if (e instanceof Error) e.message; // ✅
}

// ⑧ alwaysStrict: emits "use strict" in all output files
```

---

## 3. Module Options

```json
{
  "compilerOptions": {
    // TARGET: the JS version TypeScript compiles DOWN TO
    "target": "ES2020",
    // Options: ES3, ES5, ES6/ES2015, ES2016-ES2024, ESNext
    // Higher target = fewer polyfills needed but less browser support
    // Tip: set to ES2020+ for modern projects; bundlers transpile further if needed

    // MODULE: the module system for output files
    "module": "ESNext",
    // Options:
    //   "CommonJS"  → require/module.exports (Node.js default, jest default)
    //   "ESNext"    → import/export (for bundlers like Vite/Rollup/webpack)
    //   "NodeNext"  → Node.js ESM with .js extensions in imports
    //   "Preserve"  → keep input module syntax (TS 5.4+)

    // MODULE RESOLUTION: how TypeScript finds modules
    "moduleResolution": "bundler",
    // Options:
    //   "node"      → node_modules lookup, no .js extension required
    //   "node16"    → Node 16 ESM: requires .js extensions in relative imports
    //   "nodenext"  → Latest Node ESM behavior
    //   "bundler"   → Vite/webpack style: no .js extensions, supports package.json exports
    //   (Recommended: "bundler" for apps with bundlers, "node16/nodenext" for pure Node)

    // ES module interop
    "esModuleInterop": true,
    // Allows: import React from 'react' instead of: import * as React from 'react'
    // Also sets allowSyntheticDefaultImports: true

    "allowSyntheticDefaultImports": true,
    // Allow: import X from 'module' even if module doesn't have a default export
    // Needed for some CJS modules that export via module.exports

    // Resolve .json files as modules
    "resolveJsonModule": true,
    // import config from './config.json'; // ✅

    // Allow JS files in TypeScript project
    "allowJs": true,
    "checkJs": false, // type-check .js files too (false = just include them)

    // JSX support
    "jsx": "react-jsx"
    // Options:
    //   "preserve"     → keep JSX as-is (let Babel transform it)
    //   "react"        → transform JSX with React.createElement
    //   "react-jsx"    → use the new JSX transform (React 17+, no import needed)
    //   "react-jsxdev" → same as react-jsx but with dev-mode info
    //   "react-native" → preserve JSX (React Native bundler handles it)
  }
}
```

---

## 4. Output Options

```json
{
  "compilerOptions": {
    // WHERE TO PUT OUTPUT
    "outDir": "./dist", // compiled JS goes here
    "rootDir": "./src", // source root (mirrors structure in outDir)
    "declarationDir": "./types", // .d.ts files go here (separate from JS)

    // WHAT TO EMIT
    "declaration": true, // generate .d.ts files (required for libraries)
    "declarationMap": true, // generate .d.ts.map (enables "go to source" in editors)
    "sourceMap": true, // generate .js.map (enables debugging original TS)
    "inlineSourceMap": false, // embed source map in .js (mutually exclusive with sourceMap)
    "inlineSources": false, // embed source in source map (useful for debugging)

    // EMIT CONTROL
    "noEmit": false, // true: type-check only, emit nothing (used with separate bundler)
    "emitDeclarationOnly": false, // true: only emit .d.ts, no .js (let esbuild handle JS)
    "removeComments": false, // strip comments from output

    // INTEROP
    "importHelpers": true, // use tslib instead of inlining helpers (reduces bundle size)
    // npm install tslib (required when importHelpers: true)

    "downlevelIteration": true, // correct iteration of iterables for older targets
    // Needed if target is ES5 and you use for...of, spread, destructuring

    // SKIP TYPE CHECKING
    "skipLibCheck": true, // skip type checking of .d.ts files in node_modules
    // ✅ Almost always set this — fixes conflicts between @types packages

    "stripInternal": true // remove /** @internal */ marked declarations from .d.ts
  }
}
```

---

## 5. Type Checking Options

```json
{
  "compilerOptions": {
    // STRICTNESS (beyond strict: true)
    "noUnusedLocals": true, // error on unused local variables
    "noUnusedParameters": true, // error on unused function parameters
    "noImplicitReturns": true, // error if not all code paths return a value
    "noFallthroughCasesInSwitch": true, // error on switch case fallthrough without break

    "exactOptionalPropertyTypes": true,
    // Stricter optional properties: { x?: string } means x is string | undefined
    // NOT assignable from { x: undefined } (which would be allowed without this flag)

    "noUncheckedIndexedAccess": true,
    // arr[0] has type T | undefined, not T (safer but requires more null checks)
    // const first = arr[0]; // string | undefined (not string!)

    "noPropertyAccessFromIndexSignature": true,
    // Requires bracket notation for index signatures:
    // type Dict = { [key: string]: string };
    // dict.name;    // ❌ must use bracket notation
    // dict['name']; // ✅

    // LIBRARY TYPES
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    // Which built-in type definitions to include:
    // ES2015-ES2023: language features (Array.at, Promise.allSettled, etc.)
    // DOM: browser APIs (document, window, fetch, etc.)
    // DOM.Iterable: makes DOM collections iterable (NodeList.forEach etc.)
    // WebWorker: web worker APIs (omit DOM when writing workers)
    // ESNext: latest proposals (may be unstable)

    // TYPE ROOTS
    "types": ["node", "jest"],
    // Only include these @types packages (default: all in node_modules/@types)
    // Set this to avoid accidentally including types that shouldn't be available

    "typeRoots": ["./node_modules/@types", "./types"],
    // Where to look for type packages

    // PATHS (see Section 6)
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

---

## 6. Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"],
      "@utils/*": ["./src/utils/*"],
      "@types/*": ["./src/types/*"],
      "~/*": ["./src/*"]
    }
  }
}
```

```typescript
// Before path aliases (relative hell):
import { Button } from "../../../components/Button";
import { formatDate } from "../../../../utils/date";

// After path aliases (clean, refactor-safe):
import { Button } from "@components/Button";
import { formatDate } from "@utils/date";

// ⚠️ IMPORTANT: tsconfig paths only affect TYPE CHECKING
// They do NOT affect the actual module resolution at runtime / build time
// You must ALSO configure your bundler/runtime:

// Vite: vite.config.ts
import { resolve } from "path";
export default defineConfig({
  resolve: {
    alias: { "@": resolve(__dirname, "./src") },
  },
});

// webpack: webpack.config.js
module.exports = {
  resolve: {
    alias: { "@": path.resolve(__dirname, "src") },
  },
};

// Node.js (tsconfig-paths): ts-node -r tsconfig-paths/register
// Or use: tsx with tsconfig-paths built in
```

---

## 7. Project References

```typescript
// Project references enable:
// 1. Incremental builds (only rebuild changed packages)
// 2. Strict build ordering (referenced projects build first)
// 3. Proper type isolation between packages in a monorepo

// Monorepo structure:
// packages/
//   ui/          tsconfig.json
//   utils/       tsconfig.json
//   app/         tsconfig.json  ← references ui and utils

// packages/utils/tsconfig.json
{
  "compilerOptions": {
    "composite": true,       // required for referenced projects
    "declaration": true,     // must emit declarations
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src"]
}

// packages/ui/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../utils" }   // ui depends on utils
  ],
  "include": ["src"]
}

// packages/app/tsconfig.json
{
  "compilerOptions": {
    "outDir": "./dist"
  },
  "references": [
    { "path": "../ui" },     // app depends on ui and utils
    { "path": "../utils" }
  ],
  "include": ["src"]
}

// Build commands:
// npx tsc --build packages/app  → builds in dependency order (utils → ui → app)
// npx tsc --build --watch       → watch mode for incremental builds
// npx tsc --build --clean       → clean all build outputs

// ⚠️ "composite": true requirements:
// - All input files must be included (no "allowJs" without explicit includes)
// - "declaration": true must be set
// - "rootDir" must be set
```

---

## 8. Common Configurations by Project Type

```json5
// REACT SPA (Vite)
{
  "compilerOptions": {
    "target":              "ES2020",
    "lib":                 ["ES2020", "DOM", "DOM.Iterable"],
    "module":              "ESNext",
    "moduleResolution":    "bundler",
    "jsx":                 "react-jsx",
    "strict":              true,
    "noUnusedLocals":      true,
    "noUnusedParameters":  true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck":        true,
    "esModuleInterop":     true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule":   true,
    "isolatedModules":     true,  // required for esbuild/Vite
    "noEmit":              true   // Vite handles transpilation; TS only type-checks
  }
}

// NODE.JS SERVER (Express/Fastify)
{
  "compilerOptions": {
    "target":              "ES2022",
    "module":              "NodeNext",
    "moduleResolution":    "NodeNext",
    "lib":                 ["ES2022"],
    "outDir":              "./dist",
    "rootDir":             "./src",
    "strict":              true,
    "noUnusedLocals":      true,
    "noImplicitReturns":   true,
    "esModuleInterop":     true,
    "skipLibCheck":        true,
    "declaration":         true,
    "sourceMap":           true
  }
}

// NPM LIBRARY
{
  "compilerOptions": {
    "target":              "ES2019",  // broader compatibility
    "module":              "ESNext",
    "moduleResolution":    "bundler",
    "declaration":         true,
    "declarationMap":      true,
    "declarationDir":      "./types",
    "outDir":              "./dist",
    "rootDir":             "./src",
    "strict":              true,
    "skipLibCheck":        true,
    "composite":           true,       // for project references in consumers
    "stripInternal":       true,       // hide @internal declarations
    "importHelpers":       true        // use tslib for helpers
  }
}

// NEXT.JS
// (next.js generates its own tsconfig.json via `next build`)
// Extend it for custom settings:
{
  "extends":  "./node_modules/next/tsconfig.json",
  "compilerOptions": {
    "strict":    true,
    "baseUrl":   ".",
    "paths":     { "@/*": ["./src/*"] }
  }
}
```

---

## 9. Performance Tuning

```
SLOW TYPE CHECKING? Try these in order:

1. "skipLibCheck": true
   Skips type checking of .d.ts in node_modules.
   Almost always safe — usually the biggest single improvement.

2. "isolatedModules": true
   Each file is type-checked independently (enables parallel checking).
   Required by esbuild and Babel. Catches patterns that require cross-file info.

3. Exclude large directories:
   "exclude": ["node_modules", "dist", "**/__tests__/**", "**/*.stories.*"]

4. Use project references for monorepos:
   Only rebuild packages that changed, not the entire monorepo on every edit.

5. Incremental compilation:
   "incremental": true           // cache build info
   "tsBuildInfoFile": ".tsbuildinfo"  // where to store the cache

6. "noEmit": true + separate bundler:
   TypeScript only type-checks (fast), bundler handles transpilation (also fast).
   Use tsc --noEmit in CI for type checking, esbuild/swc/Vite for building.

7. Avoid deeply recursive types:
   Recursive conditional types are expensive. Add depth limits.

MEASURING:
   npx tsc --extendedDiagnostics    // shows time per phase
   npx tsc --generateTrace ./trace  // generates a trace for Chrome DevTools
   // Open chrome://tracing → Load → select the trace file
```

---

## 10. Migration — Adding TypeScript to an Existing Project

```
STRATEGY: incremental migration (never "big bang" rewrite)

PHASE 1: Add TypeScript infrastructure (no code changes)
  1. npm install --save-dev typescript @types/node
  2. npx tsc --init
  3. Set "allowJs": true, "checkJs": false, "strict": false
  4. Set "noEmit": true (just type-check, don't change build output)
  5. Add tsconfig include/exclude to cover your JS files
  6. Run: npx tsc --noEmit
     Fix only the errors TypeScript reports even with loose settings.

PHASE 2: Enable incremental strictness
  Add flags one at a time, fix errors, commit:
  1. "noImplicitAny": true    → most impactful, adds types to untyped code
  2. "strictNullChecks": true → catches most null/undefined bugs
  3. "strict": true           → enables all strict flags

PHASE 3: Rename files (optional, parallel to Phase 2)
  Rename .js files to .ts / .jsx files to .tsx one directory at a time.
  Update import paths as needed.
  Set "allowJs": false once all files are renamed.

PHASE 4: Add types to external dependencies
  npm install @types/react @types/lodash @types/express etc.
  For untyped packages: write minimal .d.ts files in a /typings folder.

TOOLS:
  ts-migrate (Airbnb): converts JS to TS with // @ts-ignore comments
  TypeStat: auto-fixes some TS errors with inferred types
  ts-codemods: automates some common migration patterns
```

---

## 11. Common Mistakes

### Mistake 1 — `module` and `moduleResolution` mismatch

```json
// ❌ Mismatch: ESNext modules with node resolution causes issues
{
  "module": "ESNext",
  "moduleResolution": "node" // node doesn't understand package.json exports
}

// ✅ Use matching pairs:
// For bundlers (Vite, webpack):
{ "module": "ESNext", "moduleResolution": "bundler" }

// For Node.js ESM:
{ "module": "NodeNext", "moduleResolution": "NodeNext" }

// For Node.js CJS:
{ "module": "CommonJS", "moduleResolution": "node" }
```

### Mistake 2 — `paths` aliases without bundler config

```typescript
// ❌ tsconfig paths DON'T work at runtime — only at compile time
// If you set paths in tsconfig but don't configure your bundler:
import { Button } from "@components/Button";
// TypeScript: ✅ (types found via paths)
// Runtime:    ❌ Module '@components/Button' not found

// ✅ ALWAYS set aliases in BOTH tsconfig AND your bundler
```

### Mistake 3 — Adding strict flags to a large existing codebase all at once

```
// ❌ Turning on strict: true on a large legacy codebase = hundreds of errors
// = your team spends a week fixing types instead of shipping features

// ✅ Enable flags one at a time:
// Week 1: noImplicitAny
// Week 2: strictNullChecks
// Week 3: remaining strict flags
// Fix errors in each PR before moving to the next flag
```

---

## 🔗 Related Topics

- [`01-typescript-basics.md`](./01-typescript-basics.md) — tsconfig quick start
- [`14-declaration-files.md`](./14-declaration-files.md) — `declaration`, `typeRoots`, `types`
- [`javascript-core/25-modules-and-bundling.md`](../javascript-core/25-modules-and-bundling.md) — Module systems context
- [`projects/11-component-library.md`](../projects/11-component-library.md) — Library tsconfig setup
