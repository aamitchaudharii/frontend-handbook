# 01 — TypeScript Basics

> **"TypeScript is JavaScript with a type checker sitting beside you while you code. It doesn't change what your program does at runtime — it changes what it lets you write. The value isn't in catching every possible bug; it's in making the shape of your data explicit so that when something is wrong, you find out at your editor, not in production."**

🟢 **Level: Beginner**

---

## 📚 Table of Contents

1. [What TypeScript Is (and Isn't)](#1-what-typescript-is-and-isnt)
2. [Installation and Setup](#2-installation-and-setup)
3. [Your First TypeScript File](#3-your-first-typescript-file)
4. [Type Annotations](#4-type-annotations)
5. [Type Inference](#5-type-inference)
6. [tsconfig.json Basics](#6-tsconfigjson-basics)
7. [TypeScript in the Toolchain](#7-typescript-in-the-toolchain)
8. [Common Mistakes](#8-common-mistakes)
9. [Exercises](#9-exercises)

---

## 1. What TypeScript Is (and Isn't)

```
TypeScript IS:
  ✅ A superset of JavaScript — all valid JS is valid TS
  ✅ A static type checker — catches type errors before runtime
  ✅ A compiler — transforms TS to JS (types are erased at compile time)
  ✅ A developer tool — types improve autocomplete and refactoring

TypeScript IS NOT:
  ❌ A different language that runs in the browser
  ❌ A runtime type system — types don't exist at runtime
  ❌ A guarantee of zero runtime errors — it can't type-check dynamic data
     (API responses, localStorage, user input — these need runtime validation)
  ❌ Required for correctness — but it makes correctness much easier to achieve

THE FUNDAMENTAL TRUTH:
  TypeScript types are completely erased during compilation.
  const x: number = 5; → const x = 5; (the `: number` disappears)
  At runtime, your code is plain JavaScript. TypeScript is a development-time tool.
```

---

## 2. Installation and Setup

```bash
# Install TypeScript locally (preferred)
npm install --save-dev typescript

# Or globally
npm install -g typescript

# Verify installation
npx tsc --version   # TypeScript 5.x.x

# Initialize a tsconfig.json
npx tsc --init

# Common project setups:
# Vite + React + TS:
npm create vite@latest my-app -- --template react-ts

# Next.js + TS:
npx create-next-app@latest my-app --typescript

# Node.js + TS:
npm install --save-dev typescript ts-node @types/node
```

---

## 3. Your First TypeScript File

```typescript
// hello.ts
function greet(name: string): string {
  return `Hello, ${name}!`;
}

const message = greet("Alice"); // TypeScript knows message is a string
console.log(message);

// greet(42);  ← TypeScript error: Argument of type 'number' is not assignable
//               to parameter of type 'string'.

// Compile to JavaScript:
// npx tsc hello.ts  →  produces hello.js
// npx tsc --watch   →  recompile on every file change

// hello.js (what TypeScript produces — types erased):
// function greet(name) {
//   return `Hello, ${name}!`;
// }
// const message = greet('Alice');
// console.log(message);
```

---

## 4. Type Annotations

```typescript
// Variable annotations (usually not needed — TypeScript infers them)
let name: string = "Alice";
let age: number = 30;
let active: boolean = true;

// Function parameter and return type annotations
function add(a: number, b: number): number {
  return a + b;
}

// Object type annotation
function printUser(user: { name: string; age: number }): void {
  console.log(`${user.name}, ${user.age}`);
}

// Array types
const names: string[] = ["Alice", "Bob"];
const scores: Array<number> = [95, 87, 92]; // generic syntax, equivalent

// Optional properties with ?
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}!`;
}
greet("Alice"); // "Hello, Alice!"
greet("Alice", "Hi"); // "Hi, Alice!"

// Type assertions (telling TS "I know better")
const input = document.getElementById("name") as HTMLInputElement;
input.value; // TS now knows .value exists (HTMLElement doesn't have .value)

// Non-null assertion (! — "I know this isn't null")
const el = document.getElementById("root")!; // asserts non-null
// Use sparingly — if you're wrong, it crashes at runtime with no TS warning
```

---

## 5. Type Inference

TypeScript can infer types without explicit annotations in most cases:

```typescript
// ✅ Inferred — no annotation needed
const name = "Alice"; // inferred: string
const age = 30; // inferred: number
const active = true; // inferred: boolean

const nums = [1, 2, 3]; // inferred: number[]
const user = { name: "Alice", age: 30 }; // inferred: { name: string; age: number }

// Function return type is inferred from the return statement
function double(x: number) {
  // return type inferred as number
  return x * 2;
}

// When inference works: almost always for local variables and return types
// When you SHOULD annotate explicitly:
// 1. Function parameters (TS can't infer them from nothing)
// 2. Public API surfaces (library functions, exported types)
// 3. When inference gives you a wider type than you want

// Example of inference being too wide:
const status = "active"; // inferred: string (wide)
// If you want the literal type:
const status = "active" as const; // inferred: 'active' (narrow literal)
// or:
const status: "active" | "inactive" = "active"; // explicit union
```

---

## 6. tsconfig.json Basics

```json
{
  "compilerOptions": {
    // OUTPUT
    "target": "ES2020", // JavaScript version to compile to
    "module": "ESNext", // Module system (ESNext for bundlers, CommonJS for Node)
    "outDir": "./dist", // Where to put compiled JS files
    "rootDir": "./src", // Root of your source files

    // STRICTNESS (always enable these in new projects)
    "strict": true, // Enables ALL strict checks below:
    //  "noImplicitAny": true,   // error if TS can't infer a type and falls back to `any`
    //  "strictNullChecks": true,// null/undefined are not assignable to other types
    //  "strictFunctionTypes": true,
    //  "strictBindCallApply": true,
    //  "strictPropertyInitialization": true,
    //  "noImplicitThis": true,
    //  "useUnknownInCatchVariables": true

    // MODULES
    "moduleResolution": "bundler", // "node" for Node.js, "bundler" for Vite/esbuild
    "esModuleInterop": true, // allows: import React from 'react' (instead of * as)
    "allowSyntheticDefaultImports": true,

    // TYPE CHECKING EXTRAS
    "noUnusedLocals": true, // error on unused variables
    "noUnusedParameters": true, // error on unused function parameters
    "noImplicitReturns": true, // error if not all code paths return a value
    "exactOptionalPropertyTypes": true, // stricter optional property handling

    // SOURCE MAPS (for debugging)
    "sourceMap": true,

    // PATHS (module aliases)
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"] // import from '@/components/...' instead of '../../'
    },

    // REACT
    "jsx": "react-jsx", // for React 17+ automatic JSX transform
    "lib": ["ES2020", "DOM", "DOM.Iterable"] // which type libraries to include
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 7. TypeScript in the Toolchain

```
COMMON SETUPS:

1. VITE + REACT (most common for SPAs):
   - Vite uses esbuild to strip types (no type checking during build — fast!)
   - Run tsc --noEmit separately for type checking (in CI or pre-commit)
   - tsconfig.json for editor + CI type checking
   - vite.config.ts for build configuration

2. NEXT.JS:
   - Built-in TypeScript support — just add tsconfig.json
   - Type checking runs as part of the build

3. NODE.JS:
   - ts-node: run TypeScript directly without compiling first (for scripts/CLIs)
   - tsx: faster ts-node alternative (uses esbuild)
   - For production: compile with tsc, run compiled JS

4. LIBRARIES:
   - Compile to JS + emit .d.ts declaration files
   - tsconfig: "declaration": true, "declarationDir": "./types"

THE EDITOR EXPERIENCE:
   TypeScript's most immediate value is in your editor (VS Code, WebStorm, etc.):
   ✅ Autocomplete on any typed value
   ✅ Inline error highlighting before you run anything
   ✅ Refactoring: rename symbol renames ALL usages
   ✅ "Go to definition" on any identifier, including library code
   ✅ Inline documentation from JSDoc/type comments

KEY COMMANDS:
   npx tsc           — compile all files in tsconfig.json
   npx tsc --watch   — recompile on change
   npx tsc --noEmit  — type-check only (no JS output) — use in CI
```

---

## 8. Common Mistakes

### Mistake 1 — Using `any` as the escape hatch

```typescript
// ❌ any defeats all type checking
function process(data: any) {
  return data.name.toUpperCase(); // no error even if name doesn't exist
}

// ✅ Use unknown when the type is genuinely unknown — forces you to narrow
function process(data: unknown) {
  if (typeof data === "object" && data !== null && "name" in data) {
    return (data as { name: string }).name.toUpperCase();
  }
}

// ✅ Better: define the expected shape
interface ProcessInput {
  name: string;
}
function process(data: ProcessInput) {
  return data.name.toUpperCase();
}
```

### Mistake 2 — TypeScript types for runtime validation

```typescript
// ❌ TypeScript types don't validate runtime data
interface User {
  name: string;
  age: number;
}

async function getUser(): Promise<User> {
  const res = await fetch("/api/user");
  return res.json(); // ← TypeScript trusts you, but res.json() is `any` at runtime!
  // The server could return anything — TS won't catch it
}

// ✅ Use a runtime validator for external data (zod, valibot, etc.)
import { z } from "zod";
const UserSchema = z.object({ name: z.string(), age: z.number() });
async function getUser() {
  const res = await fetch("/api/user");
  const data = await res.json();
  return UserSchema.parse(data); // throws if shape is wrong at runtime
}
```

### Mistake 3 — Ignoring strict mode

```typescript
// tsconfig without "strict": true allows many unsafe patterns:
function greet(name) {
  // parameter implicitly has 'any' type — no error!
  return name.toUpperCase();
}

// ✅ Always start projects with "strict": true
// It catches far more errors and is much harder to add later
```

---

## 9. Exercises

### Exercise 1 — Add types to an existing function

```javascript
// Add TypeScript types to this JavaScript function
function formatCurrency(amount, currency, locale) {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency: currency,
  }).format(amount);
}
```

<details>
<summary>Solution</summary>

```typescript
function formatCurrency(
  amount: number,
  currency: string,
  locale: string = "en-US",
): string {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
  }).format(amount);
}

formatCurrency(9.99, "USD"); // "US$9.99"
formatCurrency(9.99, "EUR", "de-DE"); // "9,99 €"
// formatCurrency('9.99', 'USD');     // ❌ Error: string not assignable to number
```

</details>

---

## 🔗 Related Topics

- [`02-basic-types.md`](./02-basic-types.md) — Full type system overview
- [`17-tsconfig-deep-dive.md`](./17-tsconfig-deep-dive.md) — Every compiler option explained
- [`javascript-core/25-modules-and-bundling.md`](../javascript-core/25-modules-and-bundling.md) — Module systems TypeScript uses
