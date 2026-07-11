# TypeScript — Section Index

> **A complete TypeScript curriculum from first principles to advanced type-level programming. Files are numbered in learning order. Use the level map below to jump to your starting point.**

---

## 📊 Level Map

| Level               | Files | Topics                                                                                   |
| ------------------- | ----- | ---------------------------------------------------------------------------------------- |
| 🟢 **Beginner**     | 01–04 | Setup, basic types, interfaces, functions                                                |
| 🟡 **Intermediate** | 05–10 | Unions, narrowing, generics, utility types, classes, enums                               |
| 🟠 **Advanced**     | 11–15 | Advanced generics, mapped/conditional types, template literals, declaration files, React |
| 🔴 **Senior**       | 16–17 | Advanced type patterns, tsconfig mastery                                                 |

---

## 🟢 Beginner

| File                                                                       | Topic                     | What You'll Learn                                                     |
| -------------------------------------------------------------------------- | ------------------------- | --------------------------------------------------------------------- |
| [`01-typescript-basics.md`](./01-typescript-basics.md)                     | TypeScript Basics         | What TS is, why it exists, setup, `tsc`, first config                 |
| [`02-basic-types.md`](./02-basic-types.md)                                 | Basic Types               | Primitives, arrays, tuples, `any`/`unknown`/`never`, `void`           |
| [`03-interfaces-and-type-aliases.md`](./03-interfaces-and-type-aliases.md) | Interfaces & Type Aliases | `interface` vs `type`, extending, declaration merging                 |
| [`04-functions-in-typescript.md`](./04-functions-in-typescript.md)         | Functions                 | Parameter types, return types, optional, overloads, `void` vs `never` |

---

## 🟡 Intermediate

| File                                                                         | Topic                | What You'll Learn                                                                  |
| ---------------------------------------------------------------------------- | -------------------- | ---------------------------------------------------------------------------------- |
| [`05-union-and-intersection-types.md`](./05-union-and-intersection-types.md) | Union & Intersection | `\|`, `&`, discriminated unions, exhaustiveness checking                           |
| [`06-type-narrowing.md`](./06-type-narrowing.md)                             | Type Narrowing       | `typeof`, `instanceof`, `in`, type predicates, assertion functions                 |
| [`07-generics.md`](./07-generics.md)                                         | Generics             | Generic functions, interfaces, classes, constraints, defaults                      |
| [`08-utility-types.md`](./08-utility-types.md)                               | Utility Types        | `Partial`, `Required`, `Pick`, `Omit`, `Record`, `Readonly`, `ReturnType`          |
| [`09-classes-and-access-modifiers.md`](./09-classes-and-access-modifiers.md) | Classes              | `public`/`private`/`#`, `readonly`, `abstract`, `implements`, parameter properties |
| [`10-enums-and-literal-types.md`](./10-enums-and-literal-types.md)           | Enums & Literals     | `const enum`, string/numeric enums, literal unions, `as const`                     |

---

## 🟠 Advanced

| File                                                                         | Topic                      | What You'll Learn                                                        |
| ---------------------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------ |
| [`11-advanced-generics.md`](./11-advanced-generics.md)                       | Advanced Generics          | `infer`, conditional types, variance, higher-kinded patterns             |
| [`12-mapped-and-conditional-types.md`](./12-mapped-and-conditional-types.md) | Mapped & Conditional Types | `keyof`, `in`, `as`, remapping, distributive conditional types           |
| [`13-template-literal-types.md`](./13-template-literal-types.md)             | Template Literal Types     | String manipulation at the type level, event name inference              |
| [`14-declaration-files.md`](./14-declaration-files.md)                       | Declaration Files          | `.d.ts`, `@types`, module augmentation, global augmentation              |
| [`15-typescript-with-react.md`](./15-typescript-with-react.md)               | TypeScript + React         | Props, hooks, events, generic components, `forwardRef`, `ComponentProps` |

---

## 🔴 Senior

| File                                                             | Topic                  | What You'll Learn                                                        |
| ---------------------------------------------------------------- | ---------------------- | ------------------------------------------------------------------------ |
| [`16-advanced-type-patterns.md`](./16-advanced-type-patterns.md) | Advanced Type Patterns | Branded types, builder pattern, opaque types, type-safe APIs             |
| [`17-tsconfig-deep-dive.md`](./17-tsconfig-deep-dive.md)         | tsconfig Deep Dive     | All key compiler options, strict flags, project references, path aliases |

---

## 🎯 Suggested Learning Paths

### JavaScript developer learning TypeScript

`01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10`

### TypeScript user going deeper

`05 → 06 → 07 → 08 → 11 → 12 → 13 → 15`

### Senior / type-system mastery

`11 → 12 → 13 → 14 → 16 → 17`

---

## 🔗 Related Sections

- [`javascript-core/`](../javascript-core/) — JS fundamentals TypeScript builds on
- [`patterns/`](../patterns/) — React patterns, many shown with TypeScript
- [`projects/11-component-library.md`](../projects/11-component-library.md) — TypeScript in a real library
