# 11 — Project: Component Library / Design System

> **"A component library is a product whose users are other engineers, and like any product, its success is measured by adoption, not by how clever its internals are. The hardest part is never building the Button component — it's designing an API flexible enough that fifty different teams reach for it instead of writing their own, while constrained enough that the fifty usages still look and behave consistently."**

This project guide builds a small but production-grade component library: a token-based design system, accessible primitive components (Button, Input, Select, Modal), a theming system, Storybook documentation, and the packaging/versioning concerns that make a library genuinely consumable by other projects — not just a folder of components living inside one app.

---

## 📚 What You'll Build

A component library with: a design token system (colors, spacing, typography as the single source of truth), a set of accessible primitive components built on those tokens, a theming mechanism supporting light/dark mode and brand customization, Storybook for documentation and visual testing, and proper package structure for distribution (tree-shakeable, typed, versioned).

---

## Requirements

```
FUNCTIONAL:
  - Design tokens: colors, spacing scale, typography scale, border radii,
    shadows — defined once, consumed everywhere
  - Core components: Button, Input, Select, Checkbox, Modal, Tooltip,
    Toast (each accessible, each themeable)
  - Theming: support light/dark mode, and allow consuming apps to
    override brand colors without forking the library
  - Documentation: Storybook stories for every component showing all
    variants and states

NON-FUNCTIONAL:
  - Tree-shakeable: importing one component shouldn't pull in the
    entire library's bundle
  - Fully typed (TypeScript) with good autocomplete for prop combinations
  - Accessible by default — consumers shouldn't need to add ARIA
    attributes manually for standard usage
  - Versioned with clear semantic versioning and a changelog
```

---

## Architecture Overview

```
PACKAGE STRUCTURE:
  packages/
    tokens/              (design tokens — colors, spacing, typography)
      src/index.ts
    components/           (the component library itself)
      src/
        Button/
          Button.tsx
          Button.test.tsx
          Button.stories.tsx
          index.ts
        Input/
        Modal/
        ...
      index.ts            (barrel export — but see Step 4 for tree-shaking caveat)
    theme/                (theming provider and utilities)
      src/index.ts

KEY ARCHITECTURE DECISIONS:
  1. Tokens are a SEPARATE package from components, so design tools
     (Figma plugins, style dictionaries) can consume token values
     without depending on React at all.
  2. Each component owns its own styles via CSS-in-JS or CSS Modules
     scoped to that component, avoiding global style leakage.
  3. Theming uses CSS custom properties (not just JS theme objects),
     so theme switching doesn't require a React re-render of the
     entire tree (addressed in Step 3).
```

---

## Step 1 — Design Tokens as the Foundation

```typescript
// tokens/src/index.ts — the single source of truth for all design values

export const colors = {
  // Semantic naming (what it's FOR), not just raw color naming
  // (what it LOOKS like) — this is what makes theming possible later
  primary: { 50: "#eff6ff", 500: "#3b82f6", 600: "#2563eb", 900: "#1e3a8a" },
  neutral: { 50: "#fafafa", 100: "#f5f5f5", 500: "#737373", 900: "#171717" },
  success: { 50: "#f0fdf4", 500: "#22c55e", 700: "#15803d" },
  danger: { 50: "#fef2f2", 500: "#ef4444", 700: "#b91c1c" },
  warning: { 50: "#fffbeb", 500: "#f59e0b", 700: "#b45309" },
} as const;

export const spacing = {
  xs: "4px",
  sm: "8px",
  md: "16px",
  lg: "24px",
  xl: "32px",
  "2xl": "48px",
} as const;

export const typography = {
  fontFamily: {
    sans: '"Inter", system-ui, sans-serif',
    mono: '"JetBrains Mono", monospace',
  },
  fontSize: {
    xs: "12px",
    sm: "14px",
    base: "16px",
    lg: "18px",
    xl: "20px",
    "2xl": "24px",
  },
  fontWeight: { regular: 400, medium: 500, semibold: 600, bold: 700 },
  lineHeight: { tight: 1.25, normal: 1.5, relaxed: 1.75 },
} as const;

export const radii = {
  none: "0",
  sm: "4px",
  md: "8px",
  lg: "12px",
  full: "9999px",
} as const;

export const shadows = {
  sm: "0 1px 2px rgba(0,0,0,0.05)",
  md: "0 4px 6px rgba(0,0,0,0.1)",
  lg: "0 10px 15px rgba(0,0,0,0.1)",
} as const;

// Export everything as a unified token object, for convenient consumption
export const tokens = { colors, spacing, typography, radii, shadows };
```

**Key decision:** colors are named SEMANTICALLY (`primary`, `danger`, `success`) rather than DESCRIPTIVELY (`blue`, `red`, `green`) — this is what allows a consuming application to theme the library by simply redefining what `primary` MAPS TO (a different hex value, or even a different base color entirely for a rebrand), without needing every component's internal code to change. A component that references `colors.danger.500` doesn't care whether `danger` is currently mapped to red or to a brand-specific shade of orange.

---

## Step 2 — A Primitive Component (Button), Built for Accessibility and Composition

```typescript
// components/src/Button/Button.tsx

import { forwardRef, ButtonHTMLAttributes, ElementType, ComponentPropsWithoutRef } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';

const buttonStyles = cva(
  // Base styles applied to ALL variants
  'inline-flex items-center justify-center font-medium rounded-md transition-colors ' +
  'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 ' +
  'disabled:opacity-50 disabled:pointer-events-none',
  {
    variants: {
      variant: {
        primary:   'bg-primary-600 text-white hover:bg-primary-700 focus-visible:ring-primary-500',
        secondary: 'bg-neutral-100 text-neutral-900 hover:bg-neutral-200 focus-visible:ring-neutral-400',
        danger:    'bg-danger-600 text-white hover:bg-danger-700 focus-visible:ring-danger-500',
        ghost:     'bg-transparent text-neutral-700 hover:bg-neutral-100',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4 text-base',
        lg: 'h-12 px-6 text-lg',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  }
);

interface ButtonOwnProps extends VariantProps<typeof buttonStyles> {
  isLoading?: boolean;
  leftIcon?:  React.ReactNode;
  rightIcon?: React.ReactNode;
}

type ButtonProps<E extends ElementType = 'button'> = ButtonOwnProps & {
  as?: E;
} & Omit<ComponentPropsWithoutRef<E>, keyof ButtonOwnProps | 'as'>;

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ as: Component = 'button', variant, size, isLoading, leftIcon, rightIcon, children, className, disabled, ...props }, ref) => {
    return (
      <Component
        ref={ref}
        className={cn(buttonStyles({ variant, size }), className)}
        disabled={disabled || isLoading}
        aria-busy={isLoading || undefined}
        {...props}
      >
        {isLoading
          ? <Spinner size={size === 'sm' ? 'xs' : 'sm'} aria-label="Loading" />
          : leftIcon}
        {children}
        {!isLoading && rightIcon}
      </Component>
    );
  }
);

Button.displayName = 'Button';
export { Button, buttonStyles };
export type { ButtonProps };
```

**Key decisions, in order of importance:**

1. **`forwardRef`** — any component a library ships MUST forward refs, since consuming applications routinely need `.focus()`, measurement, or integration with form libraries that require direct DOM access (see [`patterns/03-render-props-hoc.md`](../patterns/03-render-props-hoc.md) for why ref forwarding matters).

2. **The `as` prop (polymorphic component pattern)** — lets `Button` render as an `<a>` tag for navigation buttons, or a custom `Link` component from a router, while keeping all the styling and behavior consistent. Without this, consumers would need a SEPARATE `LinkButton` component duplicating all the same variant logic.

3. **`aria-busy`** during loading state — communicates the loading state to assistive technology, not just visually via the spinner.

4. **`disabled={disabled || isLoading}`** — prevents double-submission by disabling the button automatically during an async operation, a detail that consuming teams would otherwise need to remember to implement themselves on every usage.

5. **`class-variance-authority` (cva)** for variant management — keeps the mapping from props to class names declarative and centralized, rather than a sprawling series of conditional template literals that become unreadable as more variants are added.

---

## Step 3 — Theming via CSS Custom Properties (Not Just JS Theme Objects)

```typescript
// theme/src/index.ts

// Generate CSS custom properties FROM the token values
function generateCSSVariables(tokens: typeof import('@mylib/tokens').tokens) {
  const vars: Record<string, string> = {};

  Object.entries(tokens.colors).forEach(([colorName, shades]) => {
    Object.entries(shades).forEach(([shade, value]) => {
      vars[`--color-${colorName}-${shade}`] = value;
    });
  });

  Object.entries(tokens.spacing).forEach(([key, value]) => {
    vars[`--spacing-${key}`] = value;
  });

  return vars;
}

// Component styles reference CSS variables, NOT JS token values directly:
// .button-primary { background: var(--color-primary-600); }
// This means SWITCHING THEMES is just swapping which values the CSS
// variables resolve to — no React re-render of the component tree needed,
// no JS theme object threaded through Context

interface ThemeProviderProps {
  theme?: 'light' | 'dark';
  overrides?: Partial<typeof tokens.colors>; // for brand customization
  children: React.ReactNode;
}

function ThemeProvider({ theme = 'light', overrides, children }: ThemeProviderProps) {
  const cssVars = useMemo(() => {
    const baseTokens = theme === 'dark' ? darkTokens : lightTokens;
    const merged = overrides ? deepMerge(baseTokens, { colors: overrides }) : baseTokens;
    return generateCSSVariables(merged);
  }, [theme, overrides]);

  return (
    <div style={cssVars as React.CSSProperties} data-theme={theme}>
      {children}
    </div>
  );
}

// Consuming app overriding the brand color without forking the library:
<ThemeProvider overrides={{ primary: { 500: '#8b5cf6', 600: '#7c3aed' } }}>
  <App />
</ThemeProvider>
```

**Key decision:** theming via CSS custom properties means switching from light to dark mode (or applying a brand override) is purely a CSS-level change — the React component tree doesn't need to re-render AT ALL for the visual theme to update, because every component's styles reference `var(--color-primary-600)` rather than a hardcoded value or a JS variable read from Context. This is significantly more performant than a JS-theme-object-in-Context approach for large component trees, where a theme change would otherwise trigger a re-render cascade through every themed component.

---

## Step 4 — Tree-Shakeable Package Structure

```json
// package.json
{
  "name": "@mylib/components",
  "version": "1.4.0",
  "type": "module",
  "sideEffects": false,
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./Button": {
      "import": "./dist/Button/index.mjs",
      "types": "./dist/Button/index.d.ts"
    }
  },
  "files": ["dist"]
}
```

```
WHY "sideEffects": false MATTERS:
  This tells bundlers (webpack, Rollup, esbuild) that NONE of this
  package's modules have side effects when imported — meaning if a
  consumer does `import { Button } from '@mylib/components'` and never
  uses `Modal`, the bundler can safely OMIT the Modal module's code
  entirely from the final bundle.

  WITHOUT this flag (or if it's incorrectly set to `false` while some
  module DOES have side effects, like injecting global CSS), bundlers
  must conservatively assume every imported module might matter even
  if its exports are unused — defeating tree-shaking and bloating
  consumer bundles with code for components they never use.

PER-COMPONENT EXPORTS (the "./Button" entry above):
  Allows consumers who want maximum control to import directly:
  import { Button } from '@mylib/components/Button';
  This avoids the bundler even needing to ANALYZE the barrel file at
  all for tree-shaking to work, which can matter for build performance
  in very large component libraries.
```

---

## Step 5 — Accessible Modal Component (Focus Trap, Portal, Escape Handling)

```tsx
// components/src/Modal/Modal.tsx
import { createPortal } from "react-dom";
import { useFocusTrap } from "../hooks/useFocusTrap"; // shared internal hook
import { useScrollLock } from "../hooks/useScrollLock";

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string; // required, not optional — every modal needs an accessible name
  children: React.ReactNode;
  size?: "sm" | "md" | "lg" | "full";
}

function Modal({ isOpen, onClose, title, children, size = "md" }: ModalProps) {
  const modalRef = useFocusTrap(isOpen);
  useScrollLock(isOpen);

  useEffect(() => {
    if (!isOpen) return;
    function handleEscape(e: KeyboardEvent) {
      if (e.key === "Escape") onClose();
    }
    document.addEventListener("keydown", handleEscape);
    return () => document.removeEventListener("keydown", handleEscape);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div
      className="modal-overlay"
      onClick={(e) => e.target === e.currentTarget && onClose()}
    >
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        className={cn("modal-content", `modal-content--${size}`)}
        tabIndex={-1}
      >
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button onClick={onClose} aria-label="Close dialog">
            <CloseIcon />
          </button>
        </div>
        <div className="modal-body">{children}</div>
      </div>
    </div>,
    document.body,
  );
}
```

**Key decision:** `title` is a REQUIRED prop, not optional — this is a deliberate API design choice that makes it structurally impossible for a consumer to accidentally ship an inaccessible modal missing an `aria-labelledby` target. A library's job is to make the accessible path the EASY (in this case, the ONLY) path, rather than relying on every consuming team remembering to add accessibility attributes correctly on every usage.

---

## Step 6 — Storybook Documentation

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  argTypes: {
    variant: { control: 'select', options: ['primary', 'secondary', 'danger', 'ghost'] },
    size:    { control: 'select', options: ['sm', 'md', 'lg'] },
  },
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Primary: Story = { args: { variant: 'primary', children: 'Button' } };
export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: 8 }}>
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="danger">Danger</Button>
      <Button variant="ghost">Ghost</Button>
    </div>
  ),
};
export const Loading: Story = { args: { isLoading: true, children: 'Submitting...' } };
export const WithIcon: Story = { args: { leftIcon: <PlusIcon />, children: 'Add Item' } };

// Accessibility addon (storybook-addon-a11y) automatically audits each
// story against axe-core rules — catching contrast issues, missing
// labels, etc., directly in the documentation tool, before a component
// ever ships to a consuming application
```

---

## Library Design Checklist

```
☐ Design tokens are semantic (primary/danger/success), not descriptive
  (blue/red/green) — enables theming without component code changes
☐ Theming uses CSS custom properties for performant theme switching
☐ All components forward refs
☐ Components support the "as" prop or similar polymorphism where
  rendering as a different element is a common need
☐ Accessibility is structurally enforced where possible (required props
  like Modal's `title`), not just documented as a best practice
☐ Package.json correctly configured for tree-shaking (sideEffects: false,
  proper exports map)
☐ Every component has Storybook stories covering all variants and states
☐ Accessibility addon runs automated a11y audits in Storybook CI
```

---

## Extension Ideas

```
- Visual regression testing (Chromatic, Percy) integrated with Storybook
- A CLI tool to scaffold new components matching the library's conventions
- Automated changelog generation (changesets) tied to semantic versioning
- Figma token sync (push/pull design tokens between Figma and code)
- Component usage analytics (which components/props are actually used
  across consuming applications, to inform deprecation decisions)
```

---

## 🔗 Related Topics

- [`patterns/05-compound-components.md`](../patterns/05-compound-components.md) — Compound component API design used in more complex library components
- [`testing/01-unit-testing.md`](../testing/01-unit-testing.md) — Testing strategy for a component library
- [`debugging/04-error-boundaries.md`](../debugging/04-error-boundaries.md) — Error boundary patterns worth including in a library

---

<div align="center">

**`projects/` section complete!** 🎉

All 11 project files done:
`01-realtime-chat-application.md` · `02-ecommerce-product-page.md` · `03-kanban-board.md` ·
`04-markdown-editor.md` · `05-analytics-dashboard.md` · `06-infinite-scroll-gallery.md` ·
`07-multistep-form-wizard.md` · `08-authentication-system.md` · `09-notification-system.md` ·
`10-file-upload-system.md` · **`11-component-library.md`** ✓

**Remaining sections:** [`examples/`](../examples/) and [`diagrams/`](../diagrams/) →

</div>
