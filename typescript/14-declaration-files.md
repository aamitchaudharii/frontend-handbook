# 14 — Declaration Files

> **"A declaration file is TypeScript's way of describing JavaScript that it can't see the source of — third-party libraries, browser APIs, globally injected variables, or your own compiled output. Understanding them means you can type anything: an npm package that ships without types, a legacy internal library, or a custom global injected by your build system."**

🟠 **Level: Advanced**

---

## 📚 Table of Contents

1. [What Are Declaration Files?](#1-what-are-declaration-files)
2. [Writing a Basic .d.ts File](#2-writing-a-basic-dts-file)
3. [Ambient Declarations](#3-ambient-declarations)
4. [Module Declaration Files](#4-module-declaration-files)
5. [DefinitelyTyped and @types](#5-definitelytyped-and-types)
6. [Module Augmentation](#6-module-augmentation)
7. [Global Augmentation](#7-global-augmentation)
8. [Generating Declaration Files](#8-generating-declaration-files)
9. [Declaration File Patterns](#9-declaration-file-patterns)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. What Are Declaration Files?

```typescript
// Declaration files (.d.ts) contain ONLY type information — no runtime code
// They describe the shape of JavaScript modules, libraries, or APIs

// TypeScript resolves types for a module in this order:
// 1. Bundled types: package.json "types" or "typings" field
// 2. @types/package-name from DefinitelyTyped
// 3. Your own declaration files

// When you see:
import _ from "lodash"; // TypeScript knows Lodash's API
// It's because @types/lodash provides the declaration file

// Declaration files end in .d.ts and contain:
// - type aliases
// - interfaces
// - declare statements (ambient declarations)
// - module declarations
// They NEVER contain implementations — only type shapes
```

---

## 2. Writing a Basic .d.ts File

```typescript
// math.d.ts — describes a JavaScript math module
export declare function add(a: number, b: number): number;
export declare function subtract(a: number, b: number): number;
export declare const PI: number;

export declare class Vector2D {
  x: number;
  y: number;
  constructor(x: number, y: number);
  add(other: Vector2D): Vector2D;
  scale(factor: number): Vector2D;
  magnitude(): number;
}

export declare interface MathOptions {
  precision?: number;
  rounding?: "floor" | "ceil" | "round";
}

// The corresponding math.js implementation could be:
// module.exports = { add: (a, b) => a + b, ... }
// TypeScript consumers get full type safety without needing to change math.js
```

---

## 3. Ambient Declarations

```typescript
// `declare` keyword: tells TypeScript "this exists at runtime but has no source here"

// Ambient variable (e.g., injected by a bundler or build system)
declare const __DEV__: boolean;
declare const __VERSION__: string;
declare const __BUILD_TIME__: number;

// Usage (after declaring):
if (__DEV__) {
  console.log("Development mode");
}

// Ambient function
declare function require(module: string): unknown;
declare function alert(message: string): void;

// Ambient class
declare class EventEmitter {
  on(event: string, listener: (...args: any[]) => void): this;
  off(event: string, listener: (...args: any[]) => void): this;
  emit(event: string, ...args: any[]): boolean;
}

// Ambient namespace (older pattern, still common in large libraries)
declare namespace MyLib {
  function makeGreeting(name: string): string;
  let numberOfGreetings: number;
  interface GreetingSettings {
    greeting: string;
    duration?: number;
  }
}

// Usage: MyLib.makeGreeting('Alice')

// Ambient module (for packages without types)
declare module "some-untyped-package" {
  export function doSomething(x: string): number;
  export const VERSION: string;
  export default function main(): void;
}
```

---

## 4. Module Declaration Files

```typescript
// Pattern 1: ES Module declaration
// my-lib/index.d.ts
export interface Options {
  timeout: number;
  retries: number;
}

export declare function fetch(
  url: string,
  options?: Options,
): Promise<Response>;
export declare class HttpClient {
  constructor(baseUrl: string, options?: Options);
  get<T>(path: string): Promise<T>;
  post<T>(path: string, body: unknown): Promise<T>;
}
export declare const version: string;

// Pattern 2: CJS module that has both default and named exports
// legacy-lib/index.d.ts
declare function createClient(url: string): Client;
declare namespace createClient {
  interface Client {
    /* ... */
  }
  const version: string;
}
export = createClient; // CommonJS: module.exports = createClient

// Pattern 3: UMD library (works as module AND global)
// chart-lib/index.d.ts
export as namespace ChartLib; // available as global ChartLib in non-module scripts
export declare class Chart {
  /* ... */
}

// Pattern 4: Side-effect only modules
declare module "reflect-metadata" {} // sets up global metadata polyfill

// Pattern 5: CSS/image modules (tell TS what import './style.css' returns)
declare module "*.css" {
  const styles: { [className: string]: string };
  export default styles;
}
declare module "*.svg" {
  const ReactComponent: React.FC<React.SVGProps<SVGSVGElement>>;
  export default ReactComponent;
}
declare module "*.png" {
  const src: string;
  export default src;
}
declare module "*.json" {
  const value: unknown;
  export default value;
}
```

---

## 5. DefinitelyTyped and @types

```typescript
// DefinitelyTyped: community-maintained type definitions for JS libraries
// Published to npm as @types/package-name

// Install types for a library:
// npm install --save-dev @types/lodash @types/node @types/react

// TypeScript automatically finds @types packages in node_modules
// No configuration needed if "types" or "typeRoots" is not set in tsconfig

// Check if a package has types:
// 1. The package itself may include types (package.json "types" field)
//    → import and check: import X from 'X' — no error = types included
// 2. @types/X exists on npm
//    → npm install @types/X
// 3. Neither: write your own declaration file

// tsconfig.json options for type resolution:
{
  "compilerOptions": {
    "types": ["node", "jest"],  // ONLY include these @types packages
    // (default: all @types packages in node_modules/@types are included)

    "typeRoots": [              // where to look for @types
      "./node_modules/@types",
      "./custom-types"          // also look in local custom-types folder
    ]
  }
}

// Override a badly-typed @types package:
// Create local/typings/some-package/index.d.ts
// Add to tsconfig: "paths": { "some-package": ["local/typings/some-package"] }
```

---

## 6. Module Augmentation

```typescript
// Augmentation: ADD properties to an existing module's types
// without modifying the original package

// Augmenting Express to add an authenticated user to Request
// src/types/express.d.ts
import { User } from "../models/user";

declare module "express" {
  interface Request {
    user?: User; // added by auth middleware
    requestId: string; // added by logging middleware
    startTime: number;
  }
}

// Now in route handlers:
app.get("/profile", (req, res) => {
  req.user?.name; // ✅ TypeScript knows about req.user
  req.requestId; // ✅
});

// Augmenting Vue (adding global properties)
declare module "@vue/runtime-core" {
  interface ComponentCustomProperties {
    $http: AxiosInstance;
    $toast: ToastService;
    $filters: { formatDate(d: Date): string };
  }
}

// Augmenting a library to add missing overloads
declare module "some-library" {
  interface Options {
    myPlugin?: boolean; // add an option the library doesn't know about
  }
  export function create(options?: Options): Client;
}

// Augmenting to add methods to built-in prototypes (use carefully)
declare global {
  interface Array<T> {
    last(): T | undefined;
    groupBy<K extends string>(fn: (item: T) => K): Record<K, T[]>;
  }
  interface String {
    toTitleCase(): string;
  }
}
```

---

## 7. Global Augmentation

```typescript
// Add or modify GLOBAL types (window, globalThis, NodeJS.Global)

// src/types/global.d.ts
declare global {
  // Add properties to window
  interface Window {
    analytics: AnalyticsInstance;
    featureFlags: Record<string, boolean>;
    __APP_CONFIG__: AppConfig;
  }

  // Add global variables (available without import)
  var __DEV__: boolean;
  var __VERSION__: string;
  var __COMMIT_SHA__: string;

  // Extend NodeJS process.env types
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      JWT_SECRET: string;
      PORT?: string;
      NODE_ENV: "development" | "production" | "test";
    }
  }
}

// This file must be a MODULE (have at least one import/export)
// or the `declare global` block won't work correctly
export {}; // make it a module

// Usage:
window.analytics.track("page_view"); // ✅
process.env.DATABASE_URL; // ✅ string (not string | undefined)
process.env.PORT; // ✅ string | undefined

// IMPORTANT: declare global only works in MODULE files
// A file without imports/exports is a "script" — all declarations
// are automatically global, no `declare global` needed
```

---

## 8. Generating Declaration Files

```typescript
// When you build a TypeScript library, emit declaration files for consumers

// tsconfig.json for a library:
{
  "compilerOptions": {
    "declaration":        true,       // emit .d.ts files
    "declarationDir":     "./types",  // where to put them
    "declarationMap":     true,       // also emit .d.ts.map (for "go to source")
    "emitDeclarationOnly": false,     // set true if only generating types (e.g., using esbuild for JS)
    "stripInternal":      true        // omit declarations marked with @internal JSDoc
  }
}

// package.json for the library:
{
  "main":  "./dist/index.js",
  "types": "./types/index.d.ts",     // points consumers to your declaration file
  "exports": {
    ".": {
      "import":  "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types":   "./types/index.d.ts"
    }
  }
}

// @internal JSDoc annotation: excluded when stripInternal: true
/** @internal */
export function _internalHelper(): void { /* ... */ }
// This function won't appear in the generated .d.ts
```

---

## 9. Declaration File Patterns

```typescript
// PATTERN 1: Re-export everything from a source
// index.d.ts
export * from "./types/user";
export * from "./types/product";
export * from "./utils";
export { default as createClient } from "./client";

// PATTERN 2: Conditional types in declaration files
// Used heavily in library type definitions
export declare function process<T extends string>(input: T): Uppercase<T>;
export declare function process<T extends number>(input: T): `${T}`;

// PATTERN 3: Overloaded function declarations
export declare function createElement(tag: "canvas"): HTMLCanvasElement;
export declare function createElement(tag: "div"): HTMLDivElement;
export declare function createElement(tag: "input"): HTMLInputElement;
export declare function createElement(tag: string): HTMLElement;

// PATTERN 4: Class with private constructor (factory-only pattern)
export declare class Singleton {
  private constructor();
  static getInstance(): Singleton;
  run(): void;
}

// PATTERN 5: Namespace for related utilities
export declare namespace StringUtils {
  function capitalize(s: string): string;
  function truncate(s: string, maxLen: number): string;
  function slugify(s: string): string;
}

// PATTERN 6: Using `satisfies` in declaration files (TS 4.9+)
export declare const config: {
  endpoints: Record<string, string>;
  timeout: number;
};
```

---

## 10. Common Mistakes

### Mistake 1 — Forgetting `export {}` in global augmentation files

```typescript
// ❌ Without export {}, the file is a "script" — declare global is ignored
declare global {
  interface Window {
    myProp: string;
  }
}

// ✅ Adding export {} makes it a MODULE, enabling declare global
declare global {
  interface Window {
    myProp: string;
  }
}
export {}; // required
```

### Mistake 2 — Augmenting a module you didn't import

```typescript
// ❌ The string must match the EXACT module name as imported
declare module "Express" {
  // wrong — should be 'express' (lowercase)
  interface Request {
    user: User;
  }
}

// ✅ Exact string match
declare module "express" {
  interface Request {
    user?: User;
  }
}
```

### Mistake 3 — Wildcard module declarations are too broad

```typescript
// ❌ This silences ALL import errors for any string
declare module "*" {
  const value: any;
  export default value;
}

// ✅ Be specific about which file types you're declaring
declare module "*.svg" {
  const ReactComponent: React.FC<React.SVGProps<SVGSVGElement>>;
  export default ReactComponent;
}
declare module "*.module.css" {
  const styles: Record<string, string>;
  export default styles;
}
```

---

## 11. Exercises

### Exercise 1 — Type an untyped module

```typescript
// The package 'legacy-analytics' has no types. It exports:
// - function track(event: string, properties?: object): void
// - function identify(userId: string, traits?: object): void
// - function page(name: string): void
// - const VERSION: string
// - class Analytics with constructor(writeKey: string)
//
// Write a declaration file for it.
```

<details>
<summary>Solution</summary>

```typescript
// @types/legacy-analytics/index.d.ts
// OR: legacy-analytics.d.ts in your project

declare module "legacy-analytics" {
  export interface Properties {
    [key: string]: string | number | boolean | null | undefined;
  }

  export function track(event: string, properties?: Properties): void;
  export function identify(userId: string, traits?: Properties): void;
  export function page(name: string): void;
  export const VERSION: string;

  export class Analytics {
    constructor(writeKey: string);
    track(event: string, properties?: Properties): void;
    identify(userId: string, traits?: Properties): void;
    page(name: string): void;
  }
}

// Usage after declaring:
import { track, identify, Analytics } from "legacy-analytics";
track("button_clicked", { buttonId: "submit" }); // ✅
const client = new Analytics("key_123"); // ✅
```

</details>

---

## 🔗 Related Topics

- [`17-tsconfig-deep-dive.md`](./17-tsconfig-deep-dive.md) — `declaration`, `typeRoots`, `types` options
- [`javascript-core/25-modules-and-bundling.md`](../javascript-core/25-modules-and-bundling.md) — Module systems
- [`projects/11-component-library.md`](../projects/11-component-library.md) — Shipping .d.ts with a library
