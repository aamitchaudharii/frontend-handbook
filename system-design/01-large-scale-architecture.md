# 01 — Large-Scale Frontend Architecture

> **"Small apps have one problem: making things work. Large apps have a different problem: keeping things from interfering with each other. Architecture is the set of decisions that prevent a codebase from collapsing under its own weight."**

Large-scale frontend architecture is about managing complexity — organizational complexity (multiple teams, multiple codebases), technical complexity (many features, many dependencies), and change complexity (a product that evolves continuously without breaking what already works). This document covers the architectural patterns, structural decisions, and engineering disciplines that make large frontend applications maintainable.

---

## 📚 Table of Contents

1. [What Makes Frontend Architecture Hard](#1-what-makes-frontend-architecture-hard)
2. [Architectural Boundaries](#2-architectural-boundaries)
3. [Feature-Based Structure](#3-feature-based-structure)
4. [Layered Architecture](#4-layered-architecture)
5. [Dependency Rules](#5-dependency-rules)
6. [State Architecture](#6-state-architecture)
7. [API Layer Design](#7-api-layer-design)
8. [Error Boundaries and Resilience](#8-error-boundaries-and-resilience)
9. [Configuration Management](#9-configuration-management)
10. [Module Federation and Team Ownership](#10-module-federation-and-team-ownership)
11. [Performance Budgets at Scale](#11-performance-budgets-at-scale)
12. [Versioning and Backwards Compatibility](#12-versioning-and-backwards-compatibility)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. What Makes Frontend Architecture Hard

Small apps are easy. You put code wherever it fits, and it works. At scale, the same approach produces a system where:

```
Symptoms of poor large-scale architecture:

COUPLING:
  Changing the user authentication flow requires
  touching 47 files across 12 modules.

FRAGILITY:
  A new feature in the cart module breaks the
  checkout flow — nobody knows why.

PARALLELISM FAILURE:
  Team A is blocked waiting for Team B to ship
  the component they both need.

LOAD TIME REGRESSION:
  Adding a feature to the admin panel makes
  the marketing homepage 20KB heavier.

UNDERSTANDING BARRIER:
  A new engineer needs 3 weeks to understand
  enough of the codebase to make their first
  meaningful contribution.

All of these are architecture problems — not bugs.
```

### The Core Tension

```
Simplicity  ←──────────────────────────────→  Flexibility
(easy to understand, few abstractions)         (easy to change, many extension points)

Small app: maximize simplicity
  → flat structure, direct imports, inline everything

Large app: balance both
  → clear modules, explicit contracts, layered abstraction
  → but not so abstract that nothing is comprehensible
```

---

## 2. Architectural Boundaries

Boundaries define where one module ends and another begins. Well-drawn boundaries reduce coupling and enable independent development.

### Types of Boundaries

```
MODULE BOUNDARY:
  A folder or file that exposes a public API.
  Internal files are implementation details — never imported directly.

  src/features/auth/
    ├── index.ts        ← public API (only file imported by others)
    ├── AuthService.ts  ← implementation
    ├── AuthContext.tsx ← implementation
    ├── LoginForm.tsx   ← implementation
    └── hooks.ts        ← implementation

  Rule: only import from auth/index.ts, never from auth/LoginForm.tsx

LAYER BOUNDARY:
  Horizontal separation between infrastructure, domain, and UI.
  Data flow direction is strictly enforced.

  UI → Domain → Infrastructure
  (never the reverse)

TEAM BOUNDARY:
  Code owned by one team is not directly imported by another team.
  Cross-team communication happens through shared packages or events.

TECHNOLOGY BOUNDARY:
  Framework-specific code isolated from framework-agnostic business logic.
  React components don't contain pure business logic.
  Business logic doesn't import React.
```

### The Public API Pattern

```typescript
// ✅ auth/index.ts — explicitly define the public API
export { AuthProvider } from "./AuthContext";
export { useAuth } from "./hooks";
export { LoginForm } from "./LoginForm";
export type { User, Session } from "./types";

// Intentionally NOT exported:
// - AuthService (internal implementation)
// - authReducer (internal state)
// - validateCredentials (internal utility)

// External code:
import { useAuth, LoginForm } from "@/features/auth"; // ✅ public API
import { AuthService } from "@/features/auth/AuthService"; // ❌ implementation detail
```

---

## 3. Feature-Based Structure

Organize code by feature (vertical slice) rather than by technical role (horizontal layer).

### Horizontal (Technical) Structure — Doesn't Scale

```
src/
  components/   ← all components, from all features, mixed together
    Button.tsx
    UserCard.tsx
    ProductList.tsx
    CartItem.tsx
    CheckoutForm.tsx  // which of these are related? hard to tell
  services/     ← all services
    UserService.ts
    ProductService.ts
    CartService.ts
  hooks/        ← all hooks
    useUser.ts
    useProduct.ts
    useCart.ts
  types/        ← all types
    User.ts
    Product.ts
    Cart.ts
```

**Problems at scale:**

- Changes to one feature touch many directories
- Unclear ownership (who owns `components/`?)
- No natural encapsulation — everything is import-able by everyone

### Vertical (Feature) Structure — Scales Well

```
src/
  features/
    auth/
      index.ts          ← public API
      LoginForm.tsx
      AuthProvider.tsx
      useAuth.ts
      types.ts

    products/
      index.ts
      ProductList.tsx
      ProductCard.tsx
      ProductService.ts
      useProducts.ts
      types.ts

    cart/
      index.ts
      CartDrawer.tsx
      CartItem.tsx
      CartService.ts
      useCart.ts
      types.ts

    checkout/
      index.ts
      CheckoutFlow.tsx
      PaymentForm.tsx
      useCheckout.ts

  shared/               ← cross-cutting concerns only
    components/         ← truly generic UI components (Button, Modal, Input)
    hooks/              ← generic hooks (useDebounce, useLocalStorage)
    utils/              ← pure utilities (formatDate, generateId)
    types/              ← shared type definitions

  app/
    router.tsx
    App.tsx
    providers.tsx
```

**Benefits:**

- Feature cohesion: related code lives together
- Clear ownership: one team owns one or more feature directories
- Natural encapsulation: public API via index.ts
- Safe deletion: delete a feature folder without touching others

---

## 4. Layered Architecture

Within each feature (and for the whole app), layer code by concern:

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                          │
│  React components, styling, layout                          │
│  Knows about: UI state, user events                         │
│  Does NOT: contain business logic, call APIs directly       │
└─────────────────────────┬───────────────────────────────────┘
                          │ calls
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  DOMAIN LAYER                                                │
│  Business logic, state management, validation               │
│  Knows about: domain concepts (User, Order, Cart)           │
│  Does NOT: know about React, DOM, HTTP                      │
└─────────────────────────┬───────────────────────────────────┘
                          │ calls
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE LAYER                                        │
│  HTTP clients, local storage, WebSockets, analytics         │
│  Knows about: network, storage APIs                         │
│  Does NOT: know about business concepts or UI               │
└─────────────────────────────────────────────────────────────┘
```

### Concrete Implementation

```typescript
// INFRASTRUCTURE: HTTP client — knows nothing about users/business
// src/infrastructure/http.ts
export async function get<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(url, {
    ...options,
    headers: { 'Content-Type': 'application/json', ...options?.headers },
  });
  if (!response.ok) throw new HttpError(response.status, response.statusText);
  return response.json();
}

// INFRASTRUCTURE: specific API calls — knows URLs, not business logic
// src/features/auth/api.ts
import { get, post } from '@/infrastructure/http';

export const authApi = {
  login:   (creds: LoginCredentials) => post<Session>('/api/auth/login', creds),
  logout:  ()                        => post<void>('/api/auth/logout'),
  getUser: ()                        => get<User>('/api/auth/me'),
};

// DOMAIN: business logic — pure functions, no framework, no HTTP
// src/features/auth/authDomain.ts
export function canAccessAdmin(user: User): boolean {
  return user.role === 'admin' && user.isVerified;
}

export function getSessionExpiryDate(session: Session): Date {
  return new Date(session.createdAt + session.expiresIn * 1000);
}

export function isSessionExpired(session: Session): boolean {
  return getSessionExpiryDate(session) < new Date();
}

// PRESENTATION: React component — orchestrates domain + infrastructure
// src/features/auth/LoginForm.tsx
import { useCallback, useState } from 'react';
import { authApi } from './api';
import { useAuth } from './hooks';

export function LoginForm() {
  const { setSession } = useAuth();
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const handleSubmit = useCallback(async (credentials: LoginCredentials) => {
    setLoading(true);
    setError(null);
    try {
      const session = await authApi.login(credentials);
      setSession(session);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed');
    } finally {
      setLoading(false);
    }
  }, [setSession]);

  return <LoginFormUI onSubmit={handleSubmit} error={error} loading={loading} />;
}
```

---

## 5. Dependency Rules

The most important rule in layered architecture: **dependencies only point downward (or inward)**.

```
ALLOWED:
  Presentation → Domain → Infrastructure
  Feature → Shared

FORBIDDEN:
  Infrastructure → Domain (infrastructure should be generic)
  Infrastructure → Presentation (infrastructure knows nothing about UI)
  Domain → Presentation (domain has no framework dependencies)
  Feature A → Feature B's internals (only through public APIs or events)
```

### Enforcing Dependency Rules

```javascript
// ESLint rule: no-restricted-imports
// .eslintrc.js
module.exports = {
  rules: {
    "no-restricted-imports": [
      "error",
      {
        patterns: [
          // Features can't import from each other's internals
          {
            group: ["@/features/auth/!(index)"],
            message: "Import from @/features/auth/index instead",
          },
          {
            group: ["@/features/cart/!(index)"],
            message: "Import from @/features/cart/index instead",
          },
          // Domain layer can't import React
          // (enforced per directory with separate eslint config)
        ],
      },
    ],
  },
};
```

### Dependency Inversion for Testability

```typescript
// Without inversion: AuthService creates its own HTTP client
class AuthService {
  async login(creds: LoginCredentials) {
    return fetch("/api/auth/login", {
      /* ... */
    }); // hard to test
  }
}

// With inversion: HTTP client is injected
interface HttpClient {
  post<T>(url: string, data: unknown): Promise<T>;
}

class AuthService {
  constructor(private readonly http: HttpClient) {}

  async login(creds: LoginCredentials) {
    return this.http.post<Session>("/api/auth/login", creds);
  }
}

// In tests: inject a mock HTTP client
const mockHttp = { post: jest.fn().mockResolvedValue({ token: "test-token" }) };
const service = new AuthService(mockHttp);
```

---

## 6. State Architecture

State management at scale requires deliberate categorization and clear ownership.

### The Four Categories of State

```
1. SERVER STATE (remote data):
   Data fetched from APIs that lives on the server.
   Tools: TanStack Query, SWR, Apollo
   Characteristics: async, needs caching, can be stale, needs invalidation
   Examples: user profile, product catalog, order history

2. CLIENT STATE (UI state):
   Application state that lives only on the client.
   Tools: useState, useReducer, Zustand, Jotai
   Characteristics: synchronous, owned by app, no server sync
   Examples: modal open/closed, selected tab, theme preference, form values

3. URL STATE:
   State reflected in the URL (shareable, bookmarkable).
   Tools: React Router, Next.js router
   Characteristics: persists on reload, enables deep linking
   Examples: search query, current page, sort order, filters

4. FORM STATE:
   User input before submission.
   Tools: React Hook Form, Formik
   Characteristics: complex validation, temporary, converted to server state on submit
   Examples: login form, checkout form, settings form
```

### State Location Decision

```
Where should this state live?

  Needed by only one component?
    → useState / useReducer in that component

  Needed by a few nearby components?
    → Lift to nearest common ancestor

  Needed by a whole feature?
    → Feature-level context or Zustand store slice

  Needed by multiple features?
    → Global store or URL state

  Is it server data?
    → TanStack Query (cache, deduplicate, revalidate)

  Should it survive page reload?
    → URL state or localStorage
```

### Avoiding the Global Store Trap

```typescript
// ❌ Common mistake: everything in global Redux store
const store = {
  user:          { ... },  // global — OK
  products:      [ ... ],  // global — unnecessary
  selectedItem:  null,     // local UI state — shouldn't be global
  modalOpen:     false,    // local UI state — shouldn't be global
  formValues:    { ... },  // form state — shouldn't be global
  cartItems:     [ ... ],  // feature state — debatable
};

// ✅ Right tool for each state type
// Global (cross-cutting): auth, theme, cart
const globalStore = { user, theme, cart };

// Server state: TanStack Query handles caching
const { data: products } = useQuery({ queryKey: ['products'], queryFn: fetchProducts });

// Local UI state: component-level useState
const [modalOpen, setModalOpen] = useState(false);

// Form state: React Hook Form
const { register, handleSubmit } = useForm<LoginFormData>();
```

---

## 7. API Layer Design

### Centralizing API Calls

```typescript
// src/api/client.ts — base HTTP client with interceptors
import axios from "axios";

export const apiClient = axios.create({
  baseURL: process.env.VITE_API_URL,
  timeout: 10_000,
});

// Request interceptor: add auth token
apiClient.interceptors.request.use((config) => {
  const token = getAuthToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor: handle common errors
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      clearAuthToken();
      window.location.href = "/login";
    }
    if (error.response?.status === 503) {
      showMaintenanceMessage();
    }
    return Promise.reject(error);
  },
);
```

### Feature-Specific API Modules

```typescript
// src/features/products/api.ts
import { apiClient } from "@/api/client";
import type { Product, ProductFilters, PagedResponse } from "./types";

export const productsApi = {
  list: (filters: ProductFilters) =>
    apiClient
      .get<PagedResponse<Product>>("/products", { params: filters })
      .then((r) => r.data),

  get: (id: string) =>
    apiClient.get<Product>(`/products/${id}`).then((r) => r.data),

  create: (data: Partial<Product>) =>
    apiClient.post<Product>("/products", data).then((r) => r.data),

  update: (id: string, data: Partial<Product>) =>
    apiClient.patch<Product>(`/products/${id}`, data).then((r) => r.data),

  delete: (id: string) => apiClient.delete(`/products/${id}`),
};
```

### Query Key Factories

```typescript
// Centralized query key management — avoids string mismatches
// src/features/products/queryKeys.ts
export const productKeys = {
  all: () => ["products"] as const,
  lists: () => ["products", "list"] as const,
  list: (f: ProductFilters) => ["products", "list", f] as const,
  details: () => ["products", "detail"] as const,
  detail: (id: string) => ["products", "detail", id] as const,
};

// Usage — consistent, type-safe invalidation
queryClient.invalidateQueries({ queryKey: productKeys.lists() });
// Invalidates all list queries regardless of filters
```

---

## 8. Error Boundaries and Resilience

Large apps must fail gracefully. A single broken component should not crash the entire application.

```tsx
// src/shared/components/ErrorBoundary.tsx
import React, { Component, ReactNode } from "react";

interface Props {
  children: ReactNode;
  fallback: ReactNode | ((error: Error) => ReactNode);
  onError?: (error: Error, info: React.ErrorInfo) => void;
  resetKey?: unknown; // when this changes, reset the boundary
}

interface State {
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state = { error: null };

  static getDerivedStateFromProps(props: Props, state: State) {
    // Reset when resetKey changes
    if (state.error && props.resetKey !== undefined) {
      return { error: null };
    }
    return null;
  }

  static getDerivedStateFromError(error: Error) {
    return { error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Report to error monitoring
    this.props.onError?.(error, info);
    errorReporter.captureException(error, {
      extra: { componentStack: info.componentStack },
    });
  }

  render() {
    if (this.state.error) {
      const { fallback } = this.props;
      return typeof fallback === "function"
        ? fallback(this.state.error)
        : fallback;
    }
    return this.props.children;
  }
}

// Usage: wrap features independently
function App() {
  return (
    <ErrorBoundary fallback={<AppError />}>
      <Header />
      <ErrorBoundary
        fallback={({ error }) => <WidgetError error={error} />}
        resetKey={currentRoute}
      >
        <MainContent />
      </ErrorBoundary>
      <Footer />
    </ErrorBoundary>
  );
}
```

### Defensive Rendering

```typescript
// ✅ Components handle unexpected shapes gracefully
function UserCard({ user }: { user: User | null | undefined }) {
  if (!user) return <UserCardSkeleton />;

  return (
    <div className="user-card">
      <img src={user.avatarUrl ?? '/default-avatar.svg'} alt={user.name ?? 'User'} />
      <h2>{user.name ?? 'Unknown User'}</h2>
      <p>{user.email ?? '—'}</p>
    </div>
  );
}
```

---

## 9. Configuration Management

### Environment Variables

```typescript
// src/config/env.ts — validated, typed environment config
interface AppConfig {
  apiUrl: string;
  analyticsId: string;
  featureFlags: Record<string, boolean>;
  isDevelopment: boolean;
  isProduction: boolean;
  sentryDsn: string | null;
}

function validateConfig(): AppConfig {
  const required = ["VITE_API_URL"];
  const missing = required.filter((key) => !import.meta.env[key]);

  if (missing.length > 0) {
    throw new Error(
      `Missing required environment variables: ${missing.join(", ")}`,
    );
  }

  return {
    apiUrl: import.meta.env.VITE_API_URL,
    analyticsId: import.meta.env.VITE_ANALYTICS_ID ?? "",
    featureFlags: JSON.parse(import.meta.env.VITE_FEATURE_FLAGS ?? "{}"),
    isDevelopment: import.meta.env.DEV,
    isProduction: import.meta.env.PROD,
    sentryDsn: import.meta.env.VITE_SENTRY_DSN ?? null,
  };
}

// Validate at app startup — fail fast on misconfiguration
export const config = validateConfig();
```

### Feature Flags

```typescript
// src/features/featureFlags/index.ts
export type FeatureFlag =
  | 'new-checkout-flow'
  | 'ai-recommendations'
  | 'dark-mode'
  | 'live-chat';

class FeatureFlagService {
  #flags: Map<FeatureFlag, boolean>;

  constructor(initialFlags: Record<string, boolean>) {
    this.#flags = new Map(Object.entries(initialFlags) as [FeatureFlag, boolean][]);
  }

  isEnabled(flag: FeatureFlag): boolean {
    return this.#flags.get(flag) ?? false;
  }

  // Runtime override (from remote config, A/B testing)
  override(flag: FeatureFlag, value: boolean): void {
    this.#flags.set(flag, value);
  }
}

export const featureFlags = new FeatureFlagService(config.featureFlags);

// Usage
if (featureFlags.isEnabled('new-checkout-flow')) {
  return <NewCheckout />;
}
return <LegacyCheckout />;
```

---

## 10. Module Federation and Team Ownership

For very large teams, Module Federation allows independently deployable frontend modules.

### CODEOWNERS — Explicit Team Boundaries

```
# .github/CODEOWNERS

# Platform team owns shared infrastructure
/src/shared/              @platform-team
/src/infrastructure/      @platform-team
/.github/                 @platform-team

# Feature teams own their features
/src/features/auth/       @auth-team
/src/features/products/   @catalog-team
/src/features/cart/       @commerce-team
/src/features/checkout/   @commerce-team
/src/features/orders/     @fulfillment-team
```

### Shared Package Strategy

```
For large organizations with multiple frontend projects:

Option 1: Monorepo with shared packages (Nx, Turborepo)
  packages/
    @company/ui/          → design system components
    @company/utils/       → shared utilities
    @company/api-client/  → shared API client

Option 2: Published npm packages (versioned, independent)
  @company/design-system@1.4.2
  @company/auth-utils@2.1.0

Option 3: Module Federation (runtime sharing)
  Shell app loads remote modules at runtime
  Teams deploy independently
```

---

## 11. Performance Budgets at Scale

Large apps need automated performance enforcement to prevent regressions.

```javascript
// lighthouse.config.js — CI performance budget
module.exports = {
  ci: {
    assert: {
      assertions: {
        "categories:performance": ["error", { minScore: 0.8 }],
        "first-contentful-paint": ["error", { maxNumericValue: 2000 }],
        interactive: ["error", { maxNumericValue: 5000 }],
        "speed-index": ["warn", { maxNumericValue: 4000 }],
        "total-blocking-time": ["error", { maxNumericValue: 300 }],
        "cumulative-layout-shift": ["error", { maxNumericValue: 0.1 }],
        "largest-contentful-paint": ["error", { maxNumericValue: 2500 }],
        // Bundle size limits
        "resource-summary:script:size": ["error", { maxNumericValue: 200_000 }], // 200KB JS
        "resource-summary:stylesheet:size": [
          "error",
          { maxNumericValue: 50_000 },
        ], // 50KB CSS
      },
    },
  },
};
```

### Bundle Size Checks in CI

```yaml
# .github/workflows/bundle-check.yml
- name: Check bundle size
  run: |
    npm run build
    npx bundlesize

# bundlesize.config.json
{
  "files": [
    { "path": "dist/assets/index-*.js",   "maxSize": "150 kB" },
    { "path": "dist/assets/vendor-*.js",  "maxSize": "100 kB" },
    { "path": "dist/assets/index-*.css",  "maxSize": "30 kB"  }
  ]
}
```

---

## 12. Versioning and Backwards Compatibility

### API Versioning Strategy

```typescript
// Version-aware API client
const apiClient = axios.create({
  baseURL: `${config.apiUrl}/v2`, // API version pinned
});

// When API changes: version-negotiate gracefully
async function fetchUser(id: string): Promise<User> {
  try {
    return await apiClient.get<User>(`/users/${id}`).then((r) => r.data);
  } catch (error) {
    if (isVersionError(error)) {
      // Fall back to v1 endpoint for compatibility
      return legacyApiClient
        .get<UserV1>(`/users/${id}`)
        .then((r) => migrateUserV1toV2(r.data));
    }
    throw error;
  }
}
```

### Data Migration for Persistent State

```typescript
// When local storage schema changes, migrate on first load
interface AppState {
  version: number;
  data: unknown;
}

const MIGRATIONS: Record<number, (data: unknown) => unknown> = {
  1: (data: any) => ({ ...data, preferences: data.settings ?? {} }), // settings → preferences
  2: (data: any) => ({ ...data, userId: String(data.userId) }), // userId: number → string
};

function migrateState(stored: AppState): unknown {
  let { version, data } = stored;
  const CURRENT_VERSION = Object.keys(MIGRATIONS).length;

  while (version < CURRENT_VERSION) {
    version++;
    data = MIGRATIONS[version](data);
  }

  return data;
}
```

---

## 13. Good Practices

### ✅ Define and enforce module boundaries from day one

```typescript
// Start with clear public APIs — even for small features
// It's much harder to enforce boundaries retroactively
```

### ✅ Co-locate tests with their source

```
src/features/auth/
  LoginForm.tsx
  LoginForm.test.tsx      ← test right next to implementation
  LoginForm.stories.tsx   ← storybook story
  hooks.ts
  hooks.test.ts
```

### ✅ Document architectural decisions with ADRs

```markdown
# ADR-003: Use TanStack Query for server state

Date: 2024-01-15
Status: Accepted

## Context

We need consistent caching, deduplication, and invalidation for API data.

## Decision

Use TanStack Query for all server state. Don't use Redux for API data.

## Consequences

- Positive: reduced boilerplate, automatic caching, background refresh
- Negative: one more dependency, team needs to learn the API
```

### ✅ Keep the shared directory small and stable

```
// shared/ should be stable, well-tested utilities
// If something changes frequently → it belongs in a feature, not shared
// If it's specific to one feature → it belongs in that feature
```

---

## 14. Bad Practices

### ❌ Circular dependencies

```typescript
// ❌ Feature A imports from Feature B, B imports from A
// src/features/cart/index.ts
import { getUser } from "@/features/auth"; // A→B

// src/features/auth/index.ts
import { getCartCount } from "@/features/cart"; // B→A — CIRCULAR!
```

### ❌ God components

```typescript
// ❌ One component does everything
function App() {
  const [user, setUser] = useState(null);
  const [products, setProducts] = useState([]);
  const [cart, setCart] = useState([]);
  const [orders, setOrders] = useState([]);
  // ... 200 lines of mixed concerns

  return (
    // ... 300 lines of JSX mixing auth, products, cart, checkout
  );
}
```

### ❌ Deep prop drilling

```typescript
// ❌ Passing user through 8 component levels
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <UserMenu user={user}>
        <Avatar user={user} />
```

### ❌ Treating all state as global

```typescript
// ❌ modal open state in Redux — pointless global state
dispatch(openModal("deleteConfirmation"));
dispatch(closeModal("deleteConfirmation"));

// ✅ Modal state is local to the component that controls it
const [isOpen, setIsOpen] = useState(false);
```

---

## 15. Common Mistakes

### Mistake 1 — Premature abstraction

```typescript
// ❌ Abstracting before you understand the pattern
// You don't know what "generic" means until you've written it 3 times

// On first feature: write it directly
function fetchUsers() { ... }

// On second feature: copy and adapt
function fetchProducts() { ... }

// On third feature: now you see the pattern, extract it
function createFetcher<T>(endpoint: string) { ... }
```

### Mistake 2 — Breaking the dependency rule for convenience

```typescript
// ❌ Domain layer imports React for "convenience"
// domain/orderService.ts
import { useStore } from "react-redux"; // ❌ domain importing framework!

// ✅ Domain layer is framework-free — state passed as parameter
function calculateOrderTotal(items: OrderItem[], discount: Discount): number {
  // Pure function — no framework, no imports, highly testable
}
```

### Mistake 3 — Ignoring team ownership until it's too late

```
Teams A and B both modify the same central file (e.g., routes.ts).
This causes:
  - Constant merge conflicts
  - Accidental feature breakage
  - Deployments blocked waiting for the other team

Fix: define routing within each feature module.
Shell app dynamically registers routes from features.
```

---

## 16. Interview-Level Explanation

> **"How would you architect a large-scale frontend application for a team of 30+ engineers?"**

**Strong answer:**

> "The core challenge at that scale is preventing teams from stepping on each other while still allowing the product to move fast. I'd start with three decisions that everything else builds on.
>
> First, feature-based folder structure. Instead of organizing by technical role — all components together, all services together — you organize by business domain: `features/auth`, `features/cart`, `features/checkout`. Each feature directory exports a public API through its `index.ts`. Code outside the feature only imports through that index. This is enforced by ESLint rules or import linting tools. The benefit is that a team can work on their feature without knowing or touching other features.
>
> Second, layered architecture within each feature. Presentation components call domain logic, domain logic calls infrastructure. Dependencies only flow downward. This means your business logic has no React imports, which makes it trivially testable and portable. Your infrastructure code has no business concepts, which makes it reusable.
>
> Third, explicit CODEOWNERS files. Every directory has a designated owning team. Any change to that directory requires their review. This prevents silent cross-team coupling and makes ownership clear.
>
> For state, I'd categorize state into four types: server state (TanStack Query), client UI state (component useState), URL state (the router), and form state (React Hook Form). The mistake I see most often is putting everything in a global Redux store — including ephemeral UI state like modal open/closed. That creates coupling between components that shouldn't know about each other.
>
> For performance at scale, automated budgets in CI prevent regressions. Every deployment runs Lighthouse. Bundle size is checked against thresholds. If a PR adds 50KB to the initial bundle, it fails CI — the engineer has to justify it or find a way to lazy-load it instead."

---

## 17. Exercises

### Exercise 1 — Identify architectural violations

```typescript
// Find all architectural violations in this code:

// src/features/cart/CartService.ts
import React, { useContext } from "react"; // violation?
import { UserContext } from "../auth/UserContext"; // violation?
import { ProductService } from "../products/ProductService"; // violation?
import { formatCurrency } from "../checkout/utils"; // violation?
import { apiClient } from "@/infrastructure/http"; // violation?

export class CartService {
  getCartTotal(cart: Cart): string {
    return formatCurrency(
      cart.items.reduce((sum, item) => sum + item.price, 0),
    );
  }
}
```

<details>
<summary>Answer</summary>

```
Violations:

1. import React, { useContext } — CartService is a domain/infrastructure class
   Domain code should not import React (framework dependency in domain layer)

2. import { UserContext } from '../auth/UserContext'
   Imports implementation detail from auth feature (not through index.ts)
   Should be: import { getCurrentUser } from '@/features/auth'
   Or better: inject user as a parameter (dependency inversion)

3. import { ProductService } from '../products/ProductService'
   Cross-feature import of implementation detail (not through index.ts)
   Should be: import { ProductService } from '@/features/products'
   Or: inject product service via constructor

4. import { formatCurrency } from '../checkout/utils'
   Cross-feature import of utility function
   formatCurrency should be in shared/utils (it's generic)
   CartService shouldn't depend on checkout feature

5. import { apiClient } from '@/infrastructure/http' — This is OK
   Infrastructure → can import from infrastructure
   (though CartService mixing domain logic with HTTP call is questionable)

Clean version:
  CartService should:
  - Accept dependencies via constructor (user service, HTTP client)
  - Import only from @/features/auth (public API), @/infrastructure
  - Not import from React or other features' internals
  - Move formatCurrency to @/shared/utils
```

</details>

---

### Exercise 2 — Design a feature module structure

Design the complete directory structure and public API for a `notifications` feature that:

- Fetches notifications from the server
- Marks notifications as read
- Shows an unread count badge
- Supports real-time updates via WebSocket

<details>
<summary>Solution</summary>

```
src/features/notifications/
  index.ts                    ← public API

  api.ts                      ← HTTP calls
  websocket.ts                ← WebSocket connection

  types.ts                    ← Notification, NotificationPreferences

  notificationsDomain.ts      ← pure business logic
    getUnreadCount(notifications)
    markAsRead(notification)
    filterByType(notifications, type)
    sortByDate(notifications)

  NotificationsProvider.tsx   ← context + WebSocket setup
  NotificationBell.tsx        ← badge icon with count
  NotificationList.tsx        ← dropdown list of notifications
  NotificationItem.tsx        ← individual notification row

  hooks.ts
    useNotifications()        ← list + loading + error
    useUnreadCount()          ← just the count (optimized)
    useMarkAsRead()           ← mutation

  queryKeys.ts                ← TanStack Query keys

  notifications.test.ts       ← unit tests for domain logic
  NotificationBell.test.tsx   ← component integration tests

// index.ts — public API (only these are importable)
export { NotificationsProvider } from './NotificationsProvider';
export { NotificationBell }      from './NotificationBell';
export { useNotifications }      from './hooks';
export { useUnreadCount }        from './hooks';
export type { Notification }     from './types';
```

</details>

---

## 🔗 Related Topics

- [`system-design/02-feature-based-structure.md`](./02-feature-based-structure.md) — Feature structure in depth
- [`system-design/03-micro-frontends.md`](./03-micro-frontends.md) — Multi-team architecture
- [`system-design/04-state-management-design.md`](./04-state-management-design.md) — State architecture patterns
- [`architecture/01-layered-architecture.md`](../architecture/01-layered-architecture.md) — Layered architecture patterns
- [`performance/08-bundle-optimization.md`](../performance/08-bundle-optimization.md) — Code splitting for scale

---

<div align="center">

**Next:** [`system-design/02-feature-based-structure.md`](./02-feature-based-structure.md) →

</div>
