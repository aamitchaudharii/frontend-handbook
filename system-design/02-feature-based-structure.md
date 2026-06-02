# 02 — Feature-Based Structure

> **"The structure of your code is a prediction about which things will change together. If auth code and cart code change together — put them together. If they never do — separate them. Feature-based structure is a prediction that says: things within a business domain change together, and things across business domains change independently."**

Feature-based (vertical slice) architecture organizes code by business domain rather than technical role. It's the structural foundation that enables team autonomy, clear ownership, safe refactoring, and scalable codebases. This document covers how to design feature modules, what belongs where, how to handle cross-feature concerns, and how to migrate a horizontally-structured codebase to a feature-based one.

---

## 📚 Table of Contents

1. [Horizontal vs Vertical Structure](#1-horizontal-vs-vertical-structure)
2. [Anatomy of a Feature Module](#2-anatomy-of-a-feature-module)
3. [The Public API Contract](#3-the-public-api-contract)
4. [The Shared Directory](#4-the-shared-directory)
5. [Cross-Feature Communication](#5-cross-feature-communication)
6. [Nested Features and Sub-Features](#6-nested-features-and-sub-features)
7. [The App Shell Layer](#7-the-app-shell-layer)
8. [Complete Project Structure Example](#8-complete-project-structure-example)
9. [Tooling to Enforce Boundaries](#9-tooling-to-enforce-boundaries)
10. [Migrating from Horizontal to Vertical](#10-migrating-from-horizontal-to-vertical)
11. [Feature Flags and Conditional Features](#11-feature-flags-and-conditional-features)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. Horizontal vs Vertical Structure

### Horizontal (Technical Role) — Common but Problematic

```
src/
  components/
    Button.tsx
    Modal.tsx
    UserCard.tsx        ← belongs to 'users' domain
    ProductCard.tsx     ← belongs to 'products' domain
    CartItem.tsx        ← belongs to 'cart' domain
    CheckoutSummary.tsx ← belongs to 'checkout' domain

  services/
    authService.ts      ← belongs to 'auth' domain
    userService.ts      ← belongs to 'users' domain
    productService.ts   ← belongs to 'products' domain
    cartService.ts      ← belongs to 'cart' domain

  hooks/
    useAuth.ts
    useUser.ts
    useProducts.ts
    useCart.ts

  store/
    authSlice.ts
    userSlice.ts
    productSlice.ts
    cartSlice.ts

  types/
    User.ts
    Product.ts
    Cart.ts
```

**What goes wrong:**

- Adding a new feature (e.g., wishlists) requires touching 5+ directories
- No natural ownership — any team can touch any directory
- Import paths reveal nothing about feature relationships
- Deleting a feature requires hunting across all directories
- Understanding the cart feature requires reading `components/CartItem.tsx`, `services/cartService.ts`, `hooks/useCart.ts`, `store/cartSlice.ts`, `types/Cart.ts` — scattered everywhere

### Vertical (Feature Domain) — Scales Well

```
src/
  features/
    auth/               ← everything authentication
    users/              ← everything user management
    products/           ← everything product catalog
    cart/               ← everything shopping cart
    checkout/           ← everything checkout flow
    orders/             ← everything order management

  shared/               ← truly cross-cutting concerns only
  app/                  ← application shell, routing
```

**What improves:**

- Adding wishlists = create `features/wishlists/`, touch nothing else
- Clear ownership — team owns `features/checkout/`
- Deleting a feature = delete one directory
- Understanding cart = read `features/cart/`
- Import path `@/features/cart` communicates intent

---

## 2. Anatomy of a Feature Module

A well-structured feature module has a consistent internal structure:

```
features/
  cart/
    index.ts                ← PUBLIC API: the only entry point for other modules

    components/             ← React components for this feature
      CartDrawer.tsx
      CartItem.tsx
      CartSummary.tsx
      EmptyCart.tsx
      CartItemSkeleton.tsx  ← loading state

    hooks/                  ← React hooks for this feature
      useCart.ts
      useCartItem.ts
      useCartTotals.ts

    services/               ← Business logic + API calls
      CartService.ts
      cartApi.ts

    store/                  ← Local feature state (if needed)
      cartStore.ts
      cartReducer.ts

    utils/                  ← Feature-specific utilities
      cartCalculations.ts
      cartValidation.ts

    types/                  ← TypeScript types for this feature
      Cart.ts
      CartItem.ts
      CartSummary.ts

    constants/              ← Feature-specific constants
      cartConstants.ts

    __tests__/              ← Tests co-located with feature
      CartDrawer.test.tsx
      CartService.test.ts
      cartCalculations.test.ts

    CartDrawer.stories.tsx  ← Storybook stories (optional)
```

### What Belongs in a Feature

```
✅ BELONGS in a feature module:
  Components that are specific to this feature
  Hooks that exist to serve this feature
  API calls for this feature's endpoints
  State that belongs to this feature
  Types and interfaces used by this feature
  Tests for this feature

❌ DOES NOT BELONG in a feature module:
  Generic UI components (Button, Modal, Input) → shared/
  Generic utilities (formatDate, debounce) → shared/
  App-level config (routing, providers) → app/
  Cross-cutting concerns (error reporting, logging) → infrastructure/
```

---

## 3. The Public API Contract

Every feature module exports a controlled public API through its `index.ts`. Nothing else is directly importable.

```typescript
// features/cart/index.ts — the public API contract

// COMPONENTS: UI building blocks other modules can use
export { CartDrawer } from "./components/CartDrawer";
export { CartIcon } from "./components/CartIcon"; // badge for nav bar
export { MiniCart } from "./components/MiniCart"; // used in header

// HOOKS: data access for consumers
export { useCart } from "./hooks/useCart";
export { useCartCount } from "./hooks/useCartTotals"; // just the count

// ACTIONS: callable operations
export { addToCart } from "./services/CartService";
export { removeFromCart } from "./services/CartService";

// TYPES: shared type definitions
export type { Cart, CartItem, CartSummary } from "./types/Cart";

// NOT EXPORTED (implementation details):
// CartDrawer's internal CartItemList — not a public component
// cartReducer — internal state management
// cartApi — internal HTTP calls (consumers go through CartService)
// cartCalculations — internal utilities
// CART_CONSTANTS — internal constants
```

### Consuming the Public API

```typescript
// ✅ Correct: import from the public API
import { useCart, addToCart, CartDrawer } from "@/features/cart";

// ❌ Wrong: bypassing the public API (implementation detail import)
import { CartItemList } from "@/features/cart/components/CartItemList";
import { cartApi } from "@/features/cart/services/cartApi";
import { cartReducer } from "@/features/cart/store/cartReducer";
```

### Dynamic Public API (Export Conditionally)

```typescript
// features/admin/index.ts — only export if user has access
// (feature gating at import level)

export { AdminDashboard } from "./components/AdminDashboard";
// Note: gating is usually done at render time, not import time
// But this shows the intent of explicit exports

// For lazy-loaded admin features:
export const AdminDashboard = React.lazy(
  () => import("./components/AdminDashboard"),
);
```

---

## 4. The Shared Directory

`shared/` contains code that multiple features genuinely need. It must be carefully curated — everything here is a shared dependency, and changes require coordinating across consumers.

```
shared/
  components/           ← Generic, reusable UI components (design system)
    Button/
      Button.tsx
      Button.stories.tsx
      Button.test.tsx
      index.ts
    Input/
    Modal/
    Toast/
    Spinner/
    Table/
    Badge/

  hooks/                ← Generic, reusable hooks
    useDebounce.ts
    useLocalStorage.ts
    useMediaQuery.ts
    useClickOutside.ts
    useIntersectionObserver.ts

  utils/                ← Pure utility functions
    formatDate.ts
    formatCurrency.ts
    generateId.ts
    validators.ts
    deepClone.ts

  types/                ← Shared type definitions used across features
    api.ts              ← ApiResponse<T>, PaginatedResponse<T>
    common.ts           ← ID, Timestamp, Maybe<T>

  constants/            ← App-wide constants
    routes.ts
    breakpoints.ts
    apiConfig.ts
```

### The "Shared" Qualification Test

Before adding something to `shared/`, ask:

```
1. Is it used by at least two different features?
   (If only one feature uses it, it belongs in that feature)

2. Is it generic enough to be used without knowing the feature context?
   (CartSummary component is specific to cart — doesn't belong in shared)

3. Will it be stable?
   (Shared code changes require coordinating across all consumers)

4. Has the pattern been proven in a specific feature first?
   (Build in the feature, then extract to shared when the second use arises)
```

### Design System vs Shared Components

```
shared/components/ = design system primitives
  Button, Input, Modal, Toast, Badge, Table
  These have no business logic — they're visual building blocks
  Highly generic, extensively documented, carefully tested

features/*/components/ = feature-specific components
  CartItem, ProductCard, UserAvatar
  Contain business logic or domain knowledge
  Specific to one feature's needs
```

---

## 5. Cross-Feature Communication

Features must not directly import from each other's internals. Cross-feature communication happens through approved patterns.

### Pattern 1 — Public API Import (Simplest)

```typescript
// ✅ Feature B uses Feature A's public API
// features/checkout/CheckoutPage.tsx
import { useCart, CartSummary } from '@/features/cart';
import { useAuth }              from '@/features/auth';

export function CheckoutPage() {
  const { user }  = useAuth();
  const { items } = useCart();

  return (
    <div>
      <CartSummary items={items} />
      {/* checkout form using user data */}
    </div>
  );
}
```

**Use when:** Feature B genuinely composes Feature A's components or data.

### Pattern 2 — Event Bus (Decoupled)

```typescript
// features/cart/CartService.ts
import { eventBus } from "@/infrastructure/eventBus";

export async function checkout(cartId: string) {
  const order = await cartApi.checkout(cartId);

  // Publish event — don't call checkout feature directly
  eventBus.publish("cart:checked-out", { cartId, orderId: order.id });

  return order;
}

// features/orders/index.ts — subscribes to cart events
import { eventBus } from "@/infrastructure/eventBus";

eventBus.subscribe("cart:checked-out", ({ orderId }) => {
  // Navigate to order confirmation, update orders list, etc.
  orderStore.invalidate();
});
```

**Use when:** Action in Feature A triggers behavior in Feature B without Feature A knowing about Feature B.

### Pattern 3 — Shared State via Zustand Slice

```typescript
// For truly shared application state, use a shared store slice
// shared/store/appStore.ts

import { create } from "zustand";

interface AppStore {
  notifications: Notification[];
  addNotification: (n: Notification) => void;
  clearNotifications: () => void;
}

export const useAppStore = create<AppStore>((set) => ({
  notifications: [],
  addNotification: (n) =>
    set((s) => ({ notifications: [...s.notifications, n] })),
  clearNotifications: () => set({ notifications: [] }),
}));

// Any feature can use this shared store slice
// features/auth/services/authService.ts
import { useAppStore } from "@/shared/store/appStore";
useAppStore
  .getState()
  .addNotification({ type: "success", message: "Logged in!" });
```

### When to Use Each Pattern

```
Feature A has data that Feature B needs to display:
  → Public API import: import { useCartCount } from '@/features/cart'

Action in A should trigger behavior in B without A knowing about B:
  → Event bus: cartApi.checkout → publish 'cart:checked-out'

State that multiple features need read/write access to:
  → Shared store slice or root-level context

Feature B needs to customize behavior of Feature A:
  → Dependency injection / render props / context override
```

---

## 6. Nested Features and Sub-Features

Large features can be further divided into sub-features:

```
features/
  products/
    index.ts                  ← public API for entire products domain

    catalog/                  ← sub-feature: product browsing
      index.ts                ← public API for catalog
      ProductGrid.tsx
      ProductFilters.tsx
      useCatalog.ts

    product-detail/           ← sub-feature: single product view
      index.ts
      ProductDetail.tsx
      ProductImages.tsx
      RelatedProducts.tsx

    reviews/                  ← sub-feature: product reviews
      index.ts
      ReviewList.tsx
      AddReview.tsx
      useReviews.ts

    shared/                   ← shared within products domain only
      ProductCard.tsx         ← used by both catalog and detail
      ProductPrice.tsx
      types.ts

  # products/index.ts re-exports from sub-features
  export { ProductGrid }   from './catalog';
  export { ProductDetail } from './product-detail';
  export { ReviewList }    from './reviews';
```

### The Two-Level Public API

```typescript
// Sub-feature public API:
// features/products/catalog/index.ts
export { ProductGrid } from "./ProductGrid";
export { ProductFilters } from "./ProductFilters";
export { useCatalog } from "./useCatalog";

// Feature public API (re-exports + adds more):
// features/products/index.ts
export { ProductGrid, ProductFilters } from "./catalog";
export { ProductDetail } from "./product-detail";
export { ReviewList } from "./reviews";
export { ProductCard } from "./shared";
export type { Product } from "./shared/types";
```

---

## 7. The App Shell Layer

The app shell wires features together into the running application:

```
app/
  index.tsx           ← React root render
  App.tsx             ← Root component with providers
  providers.tsx       ← All context providers composed
  router.tsx          ← Route definitions
  layouts/
    MainLayout.tsx
    AuthLayout.tsx
    AdminLayout.tsx
  guards/
    AuthGuard.tsx     ← Protected route wrapper
    RoleGuard.tsx
```

```typescript
// app/router.tsx — only place that knows about all features
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';
import { AuthGuard } from './guards/AuthGuard';
import { MainLayout } from './layouts/MainLayout';
import { PageSkeleton } from '@/shared/components/Spinner';

// Lazy-load each feature page — separate chunk per route
const HomePage    = lazy(() => import('@/features/home/HomePage'));
const ProductsPage = lazy(() => import('@/features/products/ProductsPage'));
const CartPage    = lazy(() => import('@/features/cart/CartPage'));
const CheckoutPage = lazy(() => import('@/features/checkout/CheckoutPage'));
const AccountPage = lazy(() => import('@/features/account/AccountPage'));

export function AppRouter() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route element={<MainLayout />}>
          <Route path="/"           element={<HomePage />} />
          <Route path="/products/*" element={<ProductsPage />} />
          <Route path="/cart"       element={<CartPage />} />

          {/* Protected routes */}
          <Route element={<AuthGuard />}>
            <Route path="/checkout"  element={<CheckoutPage />} />
            <Route path="/account/*" element={<AccountPage />} />
          </Route>
        </Route>
      </Routes>
    </Suspense>
  );
}
```

```typescript
// app/providers.tsx — compose all providers
export function AppProviders({ children }: { children: ReactNode }) {
  const queryClient = useMemo(() => new QueryClient({
    defaultOptions: { queries: { staleTime: 5 * 60 * 1000 } },
  }), []);

  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <AuthProvider>
          <ThemeProvider>
            <ToastProvider>
              {children}
            </ToastProvider>
          </ThemeProvider>
        </AuthProvider>
      </BrowserRouter>
    </QueryClientProvider>
  );
}
```

---

## 8. Complete Project Structure Example

A complete e-commerce application:

```
src/
  ├── app/
  │   ├── index.tsx
  │   ├── App.tsx
  │   ├── providers.tsx
  │   ├── router.tsx
  │   ├── layouts/
  │   │   ├── MainLayout.tsx
  │   │   └── AuthLayout.tsx
  │   └── guards/
  │       └── AuthGuard.tsx
  │
  ├── features/
  │   ├── auth/
  │   │   ├── index.ts            ← export { AuthProvider, useAuth, LoginForm }
  │   │   ├── components/
  │   │   │   ├── LoginForm.tsx
  │   │   │   └── RegisterForm.tsx
  │   │   ├── hooks/
  │   │   │   └── useAuth.ts
  │   │   ├── services/
  │   │   │   ├── authApi.ts
  │   │   │   └── AuthService.ts
  │   │   └── types/
  │   │       └── auth.ts
  │   │
  │   ├── products/
  │   │   ├── index.ts
  │   │   ├── components/
  │   │   │   ├── ProductCard.tsx
  │   │   │   ├── ProductGrid.tsx
  │   │   │   └── ProductDetail.tsx
  │   │   ├── hooks/
  │   │   │   ├── useProducts.ts
  │   │   │   └── useProduct.ts
  │   │   ├── services/
  │   │   │   └── productsApi.ts
  │   │   └── types/
  │   │       └── product.ts
  │   │
  │   ├── cart/
  │   │   ├── index.ts
  │   │   ├── components/
  │   │   │   ├── CartDrawer.tsx
  │   │   │   ├── CartItem.tsx
  │   │   │   └── CartSummary.tsx
  │   │   ├── hooks/
  │   │   │   └── useCart.ts
  │   │   ├── store/
  │   │   │   └── cartStore.ts
  │   │   └── types/
  │   │       └── cart.ts
  │   │
  │   ├── checkout/
  │   │   ├── index.ts
  │   │   ├── components/
  │   │   │   ├── CheckoutFlow.tsx
  │   │   │   ├── ShippingForm.tsx
  │   │   │   └── PaymentForm.tsx
  │   │   ├── hooks/
  │   │   │   └── useCheckout.ts
  │   │   └── types/
  │   │       └── checkout.ts
  │   │
  │   └── orders/
  │       ├── index.ts
  │       ├── components/
  │       │   ├── OrderHistory.tsx
  │       │   └── OrderDetail.tsx
  │       └── hooks/
  │           └── useOrders.ts
  │
  ├── shared/
  │   ├── components/
  │   │   ├── Button/
  │   │   ├── Input/
  │   │   ├── Modal/
  │   │   ├── Toast/
  │   │   └── Spinner/
  │   ├── hooks/
  │   │   ├── useDebounce.ts
  │   │   └── useLocalStorage.ts
  │   ├── utils/
  │   │   ├── formatDate.ts
  │   │   └── formatCurrency.ts
  │   └── types/
  │       └── common.ts
  │
  └── infrastructure/
      ├── http/
      │   └── apiClient.ts
      ├── storage/
      │   └── localStorage.ts
      ├── analytics/
      │   └── analyticsService.ts
      └── eventBus/
          └── index.ts
```

---

## 9. Tooling to Enforce Boundaries

Structure alone doesn't enforce boundaries — tools do.

### ESLint Import Rules

```javascript
// .eslintrc.js
module.exports = {
  plugins: ["import"],
  rules: {
    // Restrict deep imports from feature internals
    "no-restricted-imports": [
      "error",
      {
        patterns: [
          {
            group: [
              "@/features/*/components/*",
              "@/features/*/hooks/*",
              "@/features/*/services/*",
              "@/features/*/store/*",
            ],
            message: "Import from the feature's index.ts public API instead",
          },
        ],
      },
    ],

    // No circular imports
    "import/no-cycle": ["error", { maxDepth: 5 }],
  },
};
```

### Dependency Cruiser

```javascript
// .dependency-cruiser.js — visualize and enforce dependency rules
module.exports = {
  forbidden: [
    {
      name: "no-feature-to-feature-internals",
      from: { path: "^src/features/([^/]+)/" },
      to: {
        path: "^src/features/([^/]+)/",
        // The feature in `to` must be different from `from`
        // AND the path must not be an index file
        pathNot: "^src/features/([^/]+)/index\\.(ts|tsx)$",
      },
    },
    {
      name: "no-shared-to-feature",
      from: { path: "^src/shared/" },
      to: { path: "^src/features/" },
      comment: "Shared code must not depend on features",
    },
    {
      name: "no-infrastructure-to-features",
      from: { path: "^src/infrastructure/" },
      to: { path: "^src/features/" },
      comment: "Infrastructure must not depend on features",
    },
  ],
};
```

### TypeScript Path Aliases

```json
// tsconfig.json — clean import paths
{
  "compilerOptions": {
    "paths": {
      "@/features/*": ["./src/features/*"],
      "@/shared/*": ["./src/shared/*"],
      "@/app/*": ["./src/app/*"],
      "@/infrastructure/*": ["./src/infrastructure/*"]
    }
  }
}
```

---

## 10. Migrating from Horizontal to Vertical

Migrating a large existing codebase requires a phased approach.

### Phase 1 — Create Boundaries Without Moving Files

```
Start with: identify features conceptually.

Create index.ts files that re-export from existing locations:
  features/cart/index.ts:
    export { CartItem }   from '../../components/CartItem';
    export { useCart }    from '../../hooks/useCart';
    export { CartService } from '../../services/cartService';

Enforce: all new imports go through the feature index.
Existing direct imports remain until Phase 2.

Goal: establish the contract. Don't break anything.
```

### Phase 2 — Move Files Into Features

```
Gradually move files into feature directories:
  components/CartItem.tsx → features/cart/components/CartItem.tsx
  hooks/useCart.ts       → features/cart/hooks/useCart.ts
  services/cartService.ts → features/cart/services/cartService.ts

Update feature index.ts to point to new locations.
Update any remaining direct imports.

Strategy: do one feature at a time.
          Keep the build green throughout.
```

### Phase 3 — Clean Up and Enforce

```
Delete the old horizontal directories (or archive them).
Enable ESLint rules to prevent regression.
Add dependency-cruiser checks to CI.

Celebrate 🎉
```

### The Strangler Fig Pattern

```typescript
// During migration: new code uses new structure,
// old code gradually replaced

// Old: components/UserProfile.tsx still exists
// New: features/profile/components/UserProfile.tsx also exists

// features/profile/index.ts
// Prefer the new location:
export { UserProfile } from "./components/UserProfile"; // new

// When old consumers update their imports to use the feature index,
// the old file becomes unreferenced and can be deleted.
```

---

## 11. Feature Flags and Conditional Features

```typescript
// Features can be enabled/disabled at the feature level
// features/beta-checkout/index.ts
import { featureFlags } from '@/infrastructure/featureFlags';

// Export nothing if feature is disabled
// (tree shaking removes it from bundle)
export const BetaCheckout = featureFlags.isEnabled('beta-checkout')
  ? lazy(() => import('./BetaCheckoutFlow'))
  : null;

// app/router.tsx
import { BetaCheckout } from '@/features/beta-checkout';
import { Checkout }     from '@/features/checkout';

<Route
  path="/checkout"
  element={BetaCheckout ? <BetaCheckout /> : <Checkout />}
/>
```

### A/B Testing at the Feature Level

```typescript
// Route users to different feature implementations
function resolveCheckoutFeature(userId: string) {
  // Consistent assignment: same user always gets same experience
  const variant = abTestingService.getVariant("checkout-v2", userId);

  return variant === "new"
    ? import("@/features/checkout-v2")
    : import("@/features/checkout");
}

// Usage in router
const CheckoutPage = lazy(() =>
  resolveCheckoutFeature(currentUser.id).then((m) => ({
    default: m.CheckoutPage,
  })),
);
```

---

## 12. Good Practices

### ✅ Start with the public API before the implementation

```typescript
// Design the index.ts first:
// "What do other modules need from this feature?"
// THEN implement to satisfy that contract

// features/notifications/index.ts — written first
export { NotificationBell } from "./components/NotificationBell";
export { NotificationToast } from "./components/NotificationToast";
export { useNotifications } from "./hooks/useNotifications";
export { useUnreadCount } from "./hooks/useNotifications";
export type { Notification } from "./types";

// Now implement each export
```

### ✅ Co-locate everything a developer needs to understand a feature

```
features/cart/
  README.md             ← "What is this feature? What problem does it solve?"
  ARCHITECTURE.md       ← "Key decisions, non-obvious patterns"
  components/
  hooks/
  __tests__/            ← Tests right next to what they test
```

### ✅ Extract to shared only after the second use

```
Don't put something in shared because you think it MIGHT be reused.
Build it in the feature.
When a second feature needs it: THEN extract.
This is the Rule of Three for shared code.
```

### ✅ Keep feature index.ts small

```typescript
// ✅ Good: exports only what other modules genuinely need
export { CartDrawer } from "./components/CartDrawer";
export { useCart } from "./hooks/useCart";
export type { CartItem } from "./types";

// ❌ Bad: "kitchen sink" index — exports everything
export * from "./components/CartDrawer";
export * from "./components/CartItem";
export * from "./components/CartSummary";
export * from "./components/EmptyCart";
export * from "./hooks/useCart";
export * from "./hooks/useCartItem";
export * from "./store/cartStore";
// This breaks encapsulation by exposing implementation details
```

---

## 13. Bad Practices

### ❌ Making `shared/` a dumping ground

```
❌ Everything ends up in shared:
  shared/components/ProductCard.tsx  ← product-specific, not generic
  shared/hooks/useCartItem.ts        ← cart-specific, not generic
  shared/utils/checkoutValidation.ts ← checkout-specific, not generic

shared/ should only contain TRULY generic code.
"I might need this somewhere else" is not enough to put it in shared.
```

### ❌ Features that are too granular

```
❌ Too many tiny features:
  features/
    button-tooltip/     ← this is a UI component, not a feature
    user-avatar/        ← this is a component, not a feature
    price-formatter/    ← this is a utility, not a feature

Features should map to business capabilities, not technical components.
```

### ❌ Features that are too coarse

```
❌ One massive feature:
  features/
    everything/         ← the entire app in one feature
      UserAuth.tsx
      ProductBrowsing.tsx
      ShoppingCart.tsx
      Checkout.tsx
      OrderHistory.tsx
      AdminPanel.tsx

Split by business domain. Each feature should be owned by one team.
```

---

## 14. Common Mistakes

### Mistake 1 — Importing across feature internals "just this once"

```typescript
// ❌ "It's just one import, it won't hurt"
// features/checkout/CheckoutPage.tsx
import { cartReducer } from "@/features/cart/store/cartReducer"; // internal!

// This creates a hidden coupling that:
// - Breaks if cart's internal implementation changes
// - Spreads: others see the pattern and copy it
// - Can't be caught automatically without tooling

// ✅ Expose what checkout needs through cart's public API
// features/cart/index.ts
export { getCartState } from "./store/cartStore"; // expose a read function
```

### Mistake 2 — No README in features

```
❌ Feature module with no documentation
  features/payment-gateway/
    index.ts
    components/...
    hooks/...
    // What is this? How does it work? What are the integration steps?

✅ Every non-trivial feature has a README
  features/payment-gateway/
    README.md
      # Payment Gateway Feature
      ## Purpose
      Integrates with Stripe and PayPal for payment processing.
      ## Key Concepts
      ## How to add a new payment method
      ## Environment variables required
```

### Mistake 3 — Multiple entry points to a feature

```typescript
// ❌ Consumers import from multiple places
import { CartDrawer } from "@/features/cart/components/CartDrawer";
import { useCart } from "@/features/cart/hooks/useCart";
import type { Cart } from "@/features/cart/types/Cart";

// This means changes to paths break consumers AND there's no clear contract.

// ✅ One import path per feature
import { CartDrawer, useCart } from "@/features/cart";
import type { Cart } from "@/features/cart";
```

---

## 15. Interview-Level Explanation

> **"How do you structure a large React application? What is feature-based architecture and why does it work better than organizing by technical role?"**

**Strong answer:**

> "Feature-based architecture organizes code by business domain rather than technical role. Instead of a flat `components/` directory containing everything from auth to checkout, you have `features/auth/`, `features/cart/`, `features/checkout/` — each containing all the components, hooks, services, and types for that domain.
>
> The principle is cohesion: things that change together should live together. When you add a new feature to the cart, you're working in `features/cart/` — you're not touching 5 separate directories. When you delete the cart feature, you delete one directory. When you want to understand the cart, you read one directory.
>
> The critical discipline is the public API. Every feature has an `index.ts` that explicitly declares what it exports. Code outside the feature imports only from that index — never from internal files. This creates a contract: the feature can refactor its internals freely as long as the public API stays stable. You enforce this with ESLint `no-restricted-imports` rules that prevent importing from feature internals.
>
> The `shared/` directory is for truly generic code — design system primitives, generic utility functions, hooks like `useDebounce`. The key discipline is not using it as a dumping ground. Something goes in `shared/` only after it's been used in two different features. This prevents premature abstraction.
>
> Cross-feature communication happens through either direct public API imports (Feature B reads Feature A's data) or an event bus (Feature A publishes an event that Feature B reacts to). The event bus is valuable when you want Feature A to trigger behavior in Feature B without A knowing B exists — like a cart checkout event that triggers an order confirmation without the cart knowing about orders.
>
> When I've migrated codebases to this structure, the most common reaction is: 'I wish we'd done this from the start.' It's much harder to enforce boundaries retroactively than to start with them."

---

## 16. Exercises

### Exercise 1 — Design a feature module

Design the complete structure for a `wishlist` feature:

- Users can save products to a wishlist
- Wishlist persists to the server
- Products page has an "Add to Wishlist" button
- There's a wishlist page showing all saved products
- The nav shows a count of wishlisted items

Define: directory structure, index.ts exports, how it communicates with the `products` feature.

<details>
<summary>Solution</summary>

```
features/
  wishlist/
    index.ts                  ← public API
    components/
      WishlistPage.tsx        ← full page component
      WishlistItem.tsx        ← individual wishlist row
      WishlistButton.tsx      ← heart icon button (used in products feature)
      WishlistCount.tsx       ← count badge for navigation
    hooks/
      useWishlist.ts          ← list + loading state
      useWishlistItem.ts      ← is item wishlisted? toggle action
    services/
      wishlistApi.ts          ← HTTP calls
    store/
      wishlistStore.ts        ← optimistic updates
    types/
      wishlist.ts             ← WishlistItem type

// index.ts — public API
export { WishlistPage }    from './components/WishlistPage';
export { WishlistButton }  from './components/WishlistButton'; // used by products
export { WishlistCount }   from './components/WishlistCount';  // used by nav
export { useWishlist }     from './hooks/useWishlist';
export { useWishlistItem } from './hooks/useWishlistItem';     // used by products
export type { WishlistItem } from './types/wishlist';

// Cross-feature communication:
// products feature uses WishlistButton and useWishlistItem from wishlist's public API
// features/products/components/ProductCard.tsx:
import { WishlistButton, useWishlistItem } from '@/features/wishlist';

// No circular dependency:
// products → wishlist (products uses wishlist's button)
// wishlist → products (wishlist page shows ProductCard from products)
// ... wait, that IS circular!

// Fix: wishlist page renders product data using only primitives
// it doesn't import ProductCard from products
// Instead: wishlist shows its own simple product representation
// OR: products exports a lightweight ProductThumbnail that wishlist can use

// Better fix: wishlist only imports from products' public API
import { ProductThumbnail } from '@/features/products';
// products explicitly exports this lightweight component for consumption by other features
```

</details>

---

### Exercise 2 — Identify structure problems

```
src/
  features/
    user/
      UserProfile.tsx
      UserSettings.tsx
      UserAvatar.tsx
      formatUserName.ts
      validateEmail.ts
      Button.tsx
      Modal.tsx
      useDebounce.ts
      UserService.ts
      ProductCard.tsx      ← ??
      useCart.ts           ← ??

    products/
      ProductList.tsx
      ProductDetail.tsx
      useProducts.ts

  shared/
    UserAvatar.tsx         ← ??
    ProductCard.tsx        ← ??
```

Identify all problems and propose fixes.

<details>
<summary>Answer</summary>

```
Problems:

1. Button.tsx in features/user/ — generic UI component
   Fix: move to shared/components/Button/

2. Modal.tsx in features/user/ — generic UI component
   Fix: move to shared/components/Modal/

3. useDebounce.ts in features/user/ — generic utility hook
   Fix: move to shared/hooks/useDebounce.ts

4. validateEmail.ts in features/user/ — could be generic
   If used only by user feature: fine to keep here
   If needed elsewhere: move to shared/utils/validators.ts

5. ProductCard.tsx in features/user/ — wrong feature!
   ProductCard belongs in features/products/
   Either products exports it, or it's in shared if truly generic
   Fix: move to features/products/components/ and export from products/index.ts

6. useCart.ts in features/user/ — wrong feature!
   Cart hooks belong in features/cart/
   Fix: remove from user feature, import from @/features/cart/

7. UserAvatar.tsx appears in BOTH features/user/ AND shared/
   Fix: pick one. If it's specific to user domain → features/user/
   If it's used by many features → shared/components/UserAvatar/
   But only keep ONE copy.

8. ProductCard.tsx in shared/ AND in features/user/ — duplicates
   Fix: single canonical location. If shared: shared/components/ProductCard
   If feature-specific: features/products/components/ProductCard

9. No index.ts in features/user/ — no public API contract
   Fix: create features/user/index.ts that exports the public API

Clean structure:
  features/user/
    index.ts
    components/
      UserProfile.tsx
      UserSettings.tsx
      UserAvatar.tsx        ← keep here if user-specific
    services/
      UserService.ts
    utils/
      formatUserName.ts
      validateEmail.ts      ← user-specific validation

  features/products/
    index.ts
    components/
      ProductList.tsx
      ProductDetail.tsx
      ProductCard.tsx       ← canonical location
    hooks/
      useProducts.ts

  shared/
    components/
      Button/
      Modal/
      UserAvatar/           ← if used by multiple features
    hooks/
      useDebounce.ts
```

</details>

---

## 🔗 Related Topics

- [`system-design/01-large-scale-architecture.md`](./01-large-scale-architecture.md) — Architectural principles
- [`system-design/03-micro-frontends.md`](./03-micro-frontends.md) — Feature isolation taken further
- [`system-design/04-state-management-design.md`](./04-state-management-design.md) — State per feature
- [`javascript-core/15-pub-sub-systems.md`](../javascript-core/15-pub-sub-systems.md) — Event bus for cross-feature communication

---

<div align="center">

**Next:** [`system-design/03-micro-frontends.md`](./03-micro-frontends.md) →

</div>
