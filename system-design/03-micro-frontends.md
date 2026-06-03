# 03 — Micro-Frontends

> **"Micro-frontends are the recognition that at a certain scale, the bottleneck isn't technology — it's coordination. When ten teams all deploy the same monolith, nobody can move fast. Micro-frontends give each team the autonomy to ship without waiting for anyone else."**

Micro-frontends extend the microservices concept to the frontend: the UI is composed of independently developed, deployed, and owned pieces. Done well, they eliminate cross-team deployment dependencies and enable genuine team autonomy. Done poorly, they create user-facing inconsistency, duplicated infrastructure costs, and performance regressions that nobody owns. This document covers when micro-frontends are the right choice, the major composition strategies, Module Federation, the hard problems (shared state, routing, design system), and the tradeoffs nobody warns you about.

---

## 📚 Table of Contents

1. [What Micro-Frontends Are](#1-what-micro-frontends-are)
2. [When to Use Micro-Frontends](#2-when-to-use-micro-frontends)
3. [Composition Strategies](#3-composition-strategies)
4. [Build-Time Integration](#4-build-time-integration)
5. [Runtime Integration — Module Federation](#5-runtime-integration--module-federation)
6. [Server-Side Composition](#6-server-side-composition)
7. [iframe Isolation](#7-iframe-isolation)
8. [Web Components for Integration](#8-web-components-for-integration)
9. [Shared Design System](#9-shared-design-system)
10. [Routing Across Micro-Frontends](#10-routing-across-micro-frontends)
11. [Cross-App Communication](#11-cross-app-communication)
12. [Shared Authentication](#12-shared-authentication)
13. [Performance Implications](#13-performance-implications)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. What Micro-Frontends Are

A micro-frontend is a self-contained, independently deployable frontend application that owns a specific business domain. Multiple micro-frontends are composed together — at build time, at runtime, or at the server — into a cohesive user experience.

```
Traditional Monolithic Frontend:
  ┌───────────────────────────────────────────────────────────────────┐
  │                    Single Frontend Application                     │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
  │  │   Auth   │  │ Products │  │   Cart   │  │    Checkout      │  │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘  │
  │                                                                     │
  │  One codebase. One deployment. All teams blocked by each other.   │
  └───────────────────────────────────────────────────────────────────┘

Micro-Frontend Architecture:
  ┌─────────────────────────────────────────────────────────────────────┐
  │                        Shell / Container App                         │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
  │  │ Auth MFE     │  │ Products MFE │  │     Checkout MFE         │  │
  │  │ (Team A)     │  │ (Team B)     │  │     (Team C)             │  │
  │  │ Deployed ✈   │  │ Deployed ✈   │  │     Deployed ✈           │  │
  │  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
  │                                                                       │
  │  Each team: independent codebase, independent CI/CD, autonomous.    │
  └─────────────────────────────────────────────────────────────────────┘
```

### The Core Value Proposition

```
ORGANIZATIONAL AUTONOMY:
  Team A ships a hotfix without coordinating with Team B.
  Team C chooses React 19 while Team A stays on React 18.
  Team D experiments with a new framework entirely.
  → Each team owns their full vertical slice from backend to frontend.

INDEPENDENT DEPLOYMENT:
  No shared release train. Team B deploys 5 times a day.
  A bug in the checkout MFE affects only checkout.
  Rollback: just roll back the checkout MFE, not the entire app.

INCREMENTAL MIGRATION:
  Migrate from AngularJS to React: one section at a time.
  The legacy code runs alongside the modern code.
  No big-bang rewrite required.
```

---

## 2. When to Use Micro-Frontends

Micro-frontends solve organizational problems. They introduce significant technical complexity. This trade is only worth it at a certain scale.

### ✅ Good Fit

```
ORGANIZATIONAL:
  ≥ 5 frontend teams all working on one application
  Teams frequently blocked by each other's deployments
  Different sections owned by different product orgs
  Sections have very different tech requirements
  Need to incrementally migrate a legacy application

TECHNICAL:
  Sections are large and complex enough to be standalone apps
  Sections have truly independent release cycles
  Different performance/security requirements per section
  (Admin vs public-facing, for example)
```

### ❌ Poor Fit

```
< 5 teams: the coordination overhead of micro-frontends
           exceeds the coordination overhead they solve.
           Use a monorepo with feature-based structure instead.

Small teams: micro-frontends require significant DevOps overhead
             (separate CI/CD, separate deployments, shell app).
             This overhead is crushing for small teams.

Tightly coupled domains: if sections must share a lot of state
                         and communicate constantly, they're not
                         really independent. The integration cost
                         exceeds the autonomy benefit.

Startups: moving fast matters more than scalability.
          Add the complexity only when the organization
          is large enough to justify it.
```

---

## 3. Composition Strategies

There are four main ways to compose micro-frontends:

```
STRATEGY 1 — Build-Time Integration:
  MFEs published as npm packages.
  Shell imports and bundles them at build time.
  Simple, but negates independent deployment.

STRATEGY 2 — Runtime Integration (Module Federation):
  MFEs expose remote modules loaded at runtime.
  Shell loads them as needed via dynamic imports.
  True independent deployment. The modern standard.

STRATEGY 3 — Server-Side Composition:
  Server (or edge function) assembles HTML from multiple MFEs.
  Best for performance (no client-side composition overhead).
  SSR micro-frontends — complex but powerful.

STRATEGY 4 — iframe Isolation:
  Each MFE runs in an iframe.
  Complete isolation — different frameworks, no shared globals.
  Simple to implement. Poor user experience (resize, routing issues).
```

---

## 4. Build-Time Integration

The simplest approach: MFEs are versioned npm packages.

```javascript
// Team B publishes: @company/products-ui@2.1.0
// Shell app's package.json:
{
  "dependencies": {
    "@company/auth-ui":      "^1.4.0",
    "@company/products-ui":  "^2.1.0",
    "@company/cart-ui":      "^3.0.0",
    "@company/checkout-ui":  "^1.2.0"
  }
}
```

```typescript
// shell/app/router.tsx
import { ProductsModule } from '@company/products-ui';
import { CartModule }     from '@company/cart-ui';
import { AuthModule }     from '@company/auth-ui';

function AppRouter() {
  return (
    <Routes>
      <Route path="/products/*" element={<ProductsModule />} />
      <Route path="/cart"       element={<CartModule />} />
    </Routes>
  );
}
```

### Build-Time Tradeoffs

```
✅ Pros:
  Simple to implement — just npm packages
  Type-safe — TypeScript works normally
  No runtime loading overhead
  Development is familiar (just importing modules)

❌ Cons:
  NOT independent deployment — changing products-ui requires:
    1. Publish new @company/products-ui version
    2. Update shell's package.json
    3. Rebuild and deploy the shell
  Shell must be rebuilt when ANY MFE changes
  Dependency conflicts: all MFEs share same React version
  Large initial bundle: all MFEs loaded upfront
```

**Verdict:** Good for a monorepo with team isolation needs, but doesn't provide true independent deployment.

---

## 5. Runtime Integration — Module Federation

Webpack Module Federation enables truly independent deployment: MFEs expose modules at runtime via URLs, loaded only when needed.

### Concepts

```
REMOTE: a micro-frontend that exposes modules
  - Has its own webpack build and deployment
  - Exposes specific components/hooks/utilities
  - Runs at its own URL (e.g., https://products.example.com)

HOST (SHELL): the container app that loads remotes
  - Knows the URLs of remotes
  - Imports remote modules via dynamic imports
  - Can define shared dependencies to avoid duplication

SHARED SCOPE: dependencies shared between host and remotes
  - React, React-DOM: shared to avoid multiple instances
  - Design system components: shared if versioned
```

### Remote Configuration (Products MFE)

```javascript
// products-mfe/webpack.config.js
const { ModuleFederationPlugin } = require("webpack").container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "productsMFE", // must be unique
      filename: "remoteEntry.js", // the manifest file

      // What this MFE exposes to consumers
      exposes: {
        "./ProductsApp": "./src/ProductsApp",
        "./ProductCard": "./src/components/ProductCard",
        "./useProducts": "./src/hooks/useProducts",
      },

      // Dependencies to share (avoid duplicating React)
      shared: {
        react: {
          singleton: true, // only one instance allowed
          requiredVersion: "^18.0.0",
        },
        "react-dom": {
          singleton: true,
          requiredVersion: "^18.0.0",
        },
        "@company/design-system": {
          singleton: true,
          requiredVersion: "^4.0.0",
        },
      },
    }),
  ],
  output: {
    publicPath: "https://products.cdn.example.com/", // must be absolute!
  },
};
```

### Shell (Host) Configuration

```javascript
// shell/webpack.config.js
const { ModuleFederationPlugin } = require("webpack").container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "shell",

      // Remote MFEs this shell consumes
      remotes: {
        productsMFE:
          "productsMFE@https://products.cdn.example.com/remoteEntry.js",
        cartMFE: "cartMFE@https://cart.cdn.example.com/remoteEntry.js",
        checkoutMFE:
          "checkoutMFE@https://checkout.cdn.example.com/remoteEntry.js",
      },

      shared: {
        react: { singleton: true, requiredVersion: "^18.0.0" },
        "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
      },
    }),
  ],
};
```

### Loading Remote Modules

```typescript
// shell/src/router.tsx
import { lazy, Suspense } from 'react';

// Dynamic imports load from remote at runtime
const ProductsApp = lazy(() => import('productsMFE/ProductsApp'));
const CartApp     = lazy(() => import('cartMFE/CartApp'));
const CheckoutApp = lazy(() => import('checkoutMFE/CheckoutApp'));

function AppRouter() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/products/*" element={<ProductsApp />} />
        <Route path="/cart"       element={<CartApp />} />
        <Route path="/checkout/*" element={<CheckoutApp />} />
      </Routes>
    </Suspense>
  );
}
```

### Vite Module Federation

```typescript
// vite.config.ts (using @originjs/vite-plugin-federation)
import federation from "@originjs/vite-plugin-federation";

export default defineConfig({
  plugins: [
    federation({
      name: "productsMFE",
      filename: "remoteEntry.js",
      exposes: {
        "./ProductsApp": "./src/ProductsApp",
      },
      shared: ["react", "react-dom"],
    }),
  ],
});
```

### Error Handling for Remote Loading Failures

```typescript
// Shell must handle MFE loading failures gracefully
function RemoteWrapper({ children, fallback }: {
  children: ReactNode;
  fallback: ReactNode;
}) {
  return (
    <ErrorBoundary
      fallback={
        <div className="mfe-error">
          <p>This section is temporarily unavailable.</p>
          {fallback}
        </div>
      }
    >
      <Suspense fallback={<PageSkeleton />}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

// Usage
<RemoteWrapper fallback={<SimpleCheckoutLink />}>
  <CheckoutApp />
</RemoteWrapper>
```

---

## 6. Server-Side Composition

The server (or CDN edge) assembles the page from multiple MFE fragments:

```
User requests: https://example.com/product/42

Composition layer (SSR/Edge):
  1. Fetch header HTML from shell-service
  2. Fetch product detail HTML from products-service
  3. Fetch recommendations HTML from recommendations-service
  4. Assemble into complete HTML response
  5. Include each MFE's script tags for hydration

Delivered to browser: complete HTML with all content
Browser: hydrates each MFE independently
```

### Edge Composition (Cloudflare Workers / Next.js Edge)

```typescript
// edge/compose.ts — runs at CDN edge
export default async function compose(request: Request): Promise<Response> {
  const url = new URL(request.url);

  // Parallel fetch from MFE services
  const [headerHTML, contentHTML, sidebarHTML] = await Promise.all([
    fetch(`https://shell.internal/header?path=${url.pathname}`).then((r) =>
      r.text(),
    ),
    fetchContent(url.pathname),
    fetch(`https://recommendations.internal/sidebar`)
      .then((r) => r.text())
      .catch(() => "<aside><!-- recommendations unavailable --></aside>"),
  ]);

  const fullHTML = `
    <!DOCTYPE html>
    <html>
      <head>
        <title>My App</title>
        <!-- Each MFE's CSS -->
        <link rel="stylesheet" href="https://shell.cdn.com/shell.css">
        <link rel="stylesheet" href="https://products.cdn.com/products.css">
      </head>
      <body>
        ${headerHTML}
        <main>${contentHTML}</main>
        ${sidebarHTML}
        <!-- Each MFE's hydration script -->
        <script src="https://shell.cdn.com/shell.js" defer></script>
        <script src="https://products.cdn.com/products.js" defer></script>
      </body>
    </html>
  `;

  return new Response(fullHTML, {
    headers: { "Content-Type": "text/html" },
  });
}
```

---

## 7. iframe Isolation

iframes provide complete isolation — each MFE runs in its own browsing context:

```html
<!-- Shell HTML -->
<div id="app-shell">
  <header>...</header>

  <!-- Each section loads in an iframe -->
  <iframe
    id="main-content"
    src="https://products.example.com/browse"
    style="width: 100%; border: none;"
    title="Product catalog"
  ></iframe>

  <footer>...</footer>
</div>
```

### iframe Communication

```javascript
// Parent (shell) → Child (MFE)
const iframe = document.getElementById("main-content");
iframe.contentWindow.postMessage(
  {
    type: "AUTH_TOKEN",
    token: userAuthToken,
  },
  "https://products.example.com",
); // origin restriction is CRITICAL

// Child (MFE) → Parent (shell)
window.parent.postMessage(
  {
    type: "CART_UPDATED",
    count: cartItemCount,
  },
  "https://shell.example.com",
); // specify expected parent origin

// Shell listens for messages
window.addEventListener("message", (event) => {
  // ALWAYS verify origin — security critical
  if (event.origin !== "https://products.example.com") return;
  if (event.data.type === "CART_UPDATED") {
    updateCartBadge(event.data.count);
  }
});
```

### iframe Tradeoffs

```
✅ Pros:
  Complete isolation (different frameworks, separate JS runtime)
  Security boundary (CSP per iframe)
  Simplest implementation for legacy migration
  No shared dependency conflicts possible

❌ Cons:
  Resize/responsive layout is awkward (must communicate height)
  Navigation is disconnected (separate browser history)
  Performance: each iframe loads its own full runtime
  Keyboard focus and accessibility are complex
  SEO: iframe content not indexed by search engines
  Printing/PDF generation is unreliable
```

**Use iframes for:** Legacy application embedding, complete isolation requirements, third-party widget embedding.
**Don't use for:** Core user journeys in modern applications.

---

## 8. Web Components for Integration

Web Components provide a framework-agnostic way to expose MFE elements:

```typescript
// products-mfe: expose ProductCard as a custom element
class ProductCardElement extends HTMLElement {
  #root: ShadowRoot;
  #reactRoot: Root | null = null;

  static get observedAttributes() {
    return ['product-id', 'show-wishlist'];
  }

  connectedCallback() {
    this.#root = this.attachShadow({ mode: 'open' });

    // Mount React inside Shadow DOM
    this.#reactRoot = createRoot(this.#root);
    this.#render();
  }

  attributeChangedCallback() {
    this.#render();
  }

  disconnectedCallback() {
    this.#reactRoot?.unmount();
  }

  #render() {
    const productId   = this.getAttribute('product-id') ?? '';
    const showWishlist = this.hasAttribute('show-wishlist');

    this.#reactRoot?.render(
      <StandaloneProviders>
        <ProductCard productId={productId} showWishlist={showWishlist} />
      </StandaloneProviders>
    );
  }
}

customElements.define('product-card', ProductCardElement);
```

```html
<!-- Any framework or plain HTML can now use the component -->
<!-- In an Angular app: -->
<product-card product-id="42" show-wishlist></product-card>

<!-- In Vue: -->
<product-card :product-id="productId"></product-card>

<!-- In plain HTML: -->
<product-card product-id="123"></product-card>
```

---

## 9. Shared Design System

Design consistency across independently deployed MFEs requires a shared design system.

### Distribution Strategy

```
Option 1: npm package (most common)
  @company/design-system published to npm/private registry
  Each MFE pins a version
  Updates require each MFE to upgrade and deploy

  + Type safety, tree shaking, standard tooling
  - Versioning coordination overhead
  - Breaking changes require coordinated upgrades

Option 2: Module Federation shared
  Design system as a shared Module Federation module
  All MFEs automatically use the same version at runtime

  + True single version — instant global updates
  - Shared singleton: one version change affects all MFEs simultaneously
  - Version mismatches cause runtime errors

Option 3: CSS custom properties (styling layer only)
  Design tokens distributed via a CSS file
  Components independently implemented per MFE

  + No JavaScript coupling
  + Visual consistency without code sharing
  - More work: each MFE implements components independently
```

### Design System as Federated Module

```javascript
// design-system/webpack.config.js
new ModuleFederationPlugin({
  name: "designSystem",
  filename: "remoteEntry.js",
  exposes: {
    "./Button": "./src/components/Button",
    "./Input": "./src/components/Input",
    "./Modal": "./src/components/Modal",
    "./tokens": "./src/tokens", // CSS custom properties
  },
  shared: {
    react: { singleton: true },
    "react-dom": { singleton: true },
  },
});

// Each MFE uses:
const Button = lazy(() => import("designSystem/Button"));
```

---

## 10. Routing Across Micro-Frontends

Routing is one of the most complex parts of micro-frontend architecture.

### The Routing Problem

```
Single-Page Application routing assumption:
  One router controls all navigation.
  Changing the URL re-renders the appropriate component.

Micro-frontend complication:
  The shell has a router.
  Each MFE has its own router.
  How do they coordinate?
```

### Strategy 1 — Shell Owns Top-Level Routes

```typescript
// Shell router: owns the first URL segment
<Routes>
  <Route path="/products/*" element={<ProductsMFE />} />  {/* products owns /products/... */}
  <Route path="/cart"       element={<CartMFE />} />
  <Route path="/checkout/*" element={<CheckoutMFE />} />
</Routes>

// Products MFE has its own nested router:
// /products/browse
// /products/:id
// /products/search?q=...
// The shell doesn't know about these sub-routes
```

### Strategy 2 — URL as Communication Channel

```typescript
// MFEs communicate navigation intent via URL changes
// Shell listens to URL changes to switch active MFE

const ROUTE_TO_MFE: Record<string, string> = {
  "/products": "productsMFE",
  "/cart": "cartMFE",
  "/checkout": "checkoutMFE",
};

// In shell: watch for route changes from any source
window.addEventListener("popstate", updateActiveMFE);
window.addEventListener("pushstate", updateActiveMFE); // custom event from MFEs

// MFEs navigate by calling:
function navigateTo(path: string) {
  window.history.pushState({}, "", path);
  window.dispatchEvent(new CustomEvent("pushstate", { detail: { path } }));
}
```

### Deep Linking and Bookmarks

```typescript
// Shell must restore the correct MFE on initial load based on URL
function App() {
  const location = useLocation();

  const activeMFE = useMemo(() => {
    const segment = location.pathname.split('/')[1];
    return ROUTE_SEGMENTS[segment] ?? 'home';
  }, [location.pathname]);

  return (
    <>
      <Header />
      <Suspense fallback={<PageSkeleton />}>
        {activeMFE === 'products'  && <ProductsApp initialPath={location.pathname} />}
        {activeMFE === 'cart'      && <CartApp />}
        {activeMFE === 'checkout'  && <CheckoutApp initialPath={location.pathname} />}
      </Suspense>
    </>
  );
}
```

---

## 11. Cross-App Communication

MFEs must communicate without creating tight coupling.

### Pub/Sub Event Bus (Recommended)

```typescript
// infrastructure/eventBus.ts — shared module (Module Federation shared or npm)
type EventMap = {
  "auth:user-logged-in": { user: User };
  "auth:user-logged-out": void;
  "cart:item-added": { productId: string; quantity: number };
  "cart:updated": { count: number; total: number };
  "checkout:completed": { orderId: string };
};

class EventBus {
  #handlers = new Map<string, Set<Function>>();

  subscribe<K extends keyof EventMap>(
    event: K,
    handler: (data: EventMap[K]) => void,
  ): () => void {
    if (!this.#handlers.has(event)) {
      this.#handlers.set(event, new Set());
    }
    this.#handlers.get(event)!.add(handler);

    return () => this.#handlers.get(event)?.delete(handler);
  }

  publish<K extends keyof EventMap>(event: K, data: EventMap[K]): void {
    this.#handlers.get(event)?.forEach((handler) => {
      try {
        handler(data);
      } catch (err) {
        console.error(`EventBus: error in ${String(event)} handler`, err);
      }
    });
  }
}

// Singleton shared across all MFEs
export const eventBus = new EventBus();
```

```typescript
// products-mfe: publishes cart events
eventBus.publish("cart:item-added", { productId: product.id, quantity: 1 });

// shell: listens and updates header cart badge
eventBus.subscribe("cart:updated", ({ count }) => {
  setCartCount(count);
});

// No direct import between products-mfe and shell
// They communicate through shared events
```

### Custom DOM Events (Framework-Agnostic)

```typescript
// For MFEs using different frameworks, DOM custom events work without sharing code
// MFE dispatches:
document.dispatchEvent(
  new CustomEvent("cart:updated", {
    bubbles: true,
    detail: { count: 3, total: 89.97 },
  }),
);

// Shell listens:
document.addEventListener("cart:updated", (event: CustomEvent) => {
  updateCartBadge(event.detail.count);
});
```

---

## 12. Shared Authentication

Authentication must be consistent across all MFEs.

### Pattern 1 — Shell Owns Auth, MFEs Trust Shell

```typescript
// Shell handles login/logout
// Provides auth context through shared event bus or URL params

// Shell passes auth context to MFEs via query params (for iframes)
const iframe = document.getElementById("products-iframe") as HTMLIFrameElement;
iframe.src = `https://products.example.com/?token=${encodeURIComponent(authToken)}`;

// Or via postMessage after iframe loads:
iframe.contentWindow?.postMessage(
  {
    type: "AUTH_CONTEXT",
    user: currentUser,
    token: authToken,
  },
  "https://products.example.com",
);
```

### Pattern 2 — Shared Cookie/Token Storage

```typescript
// HttpOnly cookie: browser sends it with every request to same domain
// Most seamless: *.example.com cookies work for all subdomains

// Auth MFE sets cookie at /api/auth/login:
Set-Cookie: session=...; Domain=.example.com; HttpOnly; Secure; SameSite=Strict

// products.example.com — automatically receives cookie on every request
// No token sharing code needed — cookies do it

// For cross-origin MFEs (different domains): JWT in localStorage + event bus
```

### Pattern 3 — Auth Proxy MFE

```typescript
// One shared auth MFE handles all auth UI
// All MFEs load it and register for auth events

// shell mounts AuthMFE (handles login/logout UI):
<AuthMFE
  onLogin={(user) => eventBus.publish('auth:user-logged-in', { user })}
  onLogout={() => eventBus.publish('auth:user-logged-out', undefined)}
/>

// All other MFEs subscribe:
eventBus.subscribe('auth:user-logged-in', ({ user }) => setCurrentUser(user));
eventBus.subscribe('auth:user-logged-out', () => clearCurrentUser());
```

---

## 13. Performance Implications

Micro-frontends introduce performance costs that must be managed:

### Bundle Duplication

```
Without Module Federation (each MFE has own React):
  Shell:    React (42KB) + shell code (80KB) = 122KB
  Products: React (42KB) + products code (150KB) = 192KB
  Cart:     React (42KB) + cart code (60KB) = 102KB
  Total downloaded: 416KB (React downloaded 3 times!)

With Module Federation (React shared):
  Shell:    React (42KB) + shell code (80KB) = 122KB
  Products: products code only (150KB) = 150KB
  Cart:     cart code only (60KB) = 60KB
  Total downloaded: 332KB (React downloaded once)
```

### Waterfall Loading

```
Problem: MFE loading can create waterfalls

Without optimization:
  t=0:    Shell HTML loads
  t=200:  Shell JS loads → discovers it needs ProductsMFE
  t=200:  Request for remoteEntry.js (products)
  t=400:  remoteEntry.js received → discovers products.js needed
  t=400:  Request for products.js
  t=600:  products.js loaded → page interactive

With preloading:
  Shell HTML includes: <link rel="preload" href="https://products.cdn.com/remoteEntry.js" as="script">
  t=0:    Shell HTML loads → preload hints dispatched immediately
  t=200:  Shell JS + remoteEntry.js arrive simultaneously
  t=400:  products.js loaded → page interactive
  Saved: ~200ms (entire RTT)
```

```html
<!-- Shell HTML: preload critical MFE remotes -->
<link
  rel="preload"
  href="https://products.cdn.example.com/remoteEntry.js"
  as="script"
  crossorigin
/>
<link
  rel="prefetch"
  href="https://cart.cdn.example.com/remoteEntry.js"
  as="script"
  crossorigin
/>
```

---

## 14. Good Practices

### ✅ Define a micro-frontend contract (API surface)

```typescript
// Each MFE publishes its integration contract
// products-mfe/CONTRACT.md or CONTRACT.ts

export interface ProductsMFEContract {
  // Required: provided by shell
  provides: {
    authToken: string;
    theme: "light" | "dark";
    locale: string;
  };

  // Events published by this MFE
  publishes: {
    "cart:item-added": { productId: string; quantity: number };
  };

  // Events consumed by this MFE
  subscribes: {
    "auth:user-logged-in": { user: User };
    "theme:changed": { theme: "light" | "dark" };
  };
}
```

### ✅ Independent CI/CD per MFE

```yaml
# products-mfe/.github/workflows/deploy.yml
name: Deploy Products MFE
on:
  push:
    branches: [main]
    paths: ["products-mfe/**"] # only trigger when this MFE changes

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd products-mfe && npm ci && npm run build
      - run: aws s3 sync dist/ s3://products-cdn/
      - run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CDN_ID }} --paths "/*"
```

### ✅ Graceful degradation when a remote fails

```typescript
async function loadRemoteMFE(remoteName: string) {
  try {
    return await import(`${remoteName}/App`);
  } catch (error) {
    console.error(`Failed to load ${remoteName}:`, error);
    // Return a placeholder that doesn't break the shell
    return { default: FallbackComponent };
  }
}
```

---

## 15. Bad Practices

### ❌ Micro-frontends for small teams

```
A team of 3 engineers maintaining a single product
does not benefit from micro-frontends.

The overhead:
  - Multiple repos / deployments to manage
  - Shell + multiple MFE infrastructure
  - Cross-repo coordination for shared concerns
  - Multiple CI/CD pipelines

Far exceeds the benefit when everyone talks to everyone daily anyway.
```

### ❌ Sharing too much state between MFEs

```typescript
// ❌ MFEs directly accessing shared Redux store
// This couples all MFEs to the same state shape
// Any store refactoring breaks all MFEs

import { store } from "@company/shared-store";
// products-mfe, cart-mfe, checkout-mfe all depend on this

// ✅ MFEs communicate through events and their own local state
// Only the data that truly crosses boundaries goes through the event bus
```

### ❌ Different versions of the design system per MFE

```
Products MFE: @company/design-system@3.2
Cart MFE:     @company/design-system@4.0

User experience:
  Buttons look different in cart vs products
  Forms have different validation styles
  Colors slightly different between sections

Fix: design system as Module Federation singleton (one version at runtime)
Or: coordinated upgrades with strong backwards compatibility guarantees
```

---

## 16. Common Mistakes

### Mistake 1 — Not handling remote loading failures

```typescript
// ❌ No error handling: entire shell crashes if one MFE fails to load
<ProductsApp /> // if this throws, React error boundary catches it, but...

// ✅ Dedicated error boundary per MFE + fallback content
<ErrorBoundary
  fallback={
    <div>
      Product catalog is temporarily unavailable.
      <a href="/products-static">Browse cached catalog →</a>
    </div>
  }
>
  <Suspense fallback={<ProductsSkeleton />}>
    <ProductsApp />
  </Suspense>
</ErrorBoundary>
```

### Mistake 2 — MFEs that communicate too frequently

```typescript
// ❌ Tight communication coupling — MFEs aren't truly independent
productsMFE.subscribe("filter:changed", updateCart); // makes no sense
cartMFE.subscribe("product:hovered", trackAnalytics); // wrong abstraction

// If MFEs need to communicate this frequently, they may not be the right boundary
// Consider: are these two features really one feature that was split artificially?
```

### Mistake 3 — Inconsistent error handling across MFEs

```typescript
// ❌ Each MFE handles errors differently
// Products MFE: shows toast
// Cart MFE: shows modal
// Checkout MFE: inline error

// ✅ Shared error handling via event bus + shell-level toast
// Any MFE can publish an error event
eventBus.publish("app:error", {
  severity: "error",
  message: "Failed to add item to cart",
  retry: () => addToCart(productId),
});

// Shell handles all error presentation consistently
eventBus.subscribe("app:error", (error) => {
  toastService.show(error);
});
```

---

## 17. Interview-Level Explanation

> **"What are micro-frontends? When would you use them, and what are the tradeoffs?"**

**Strong answer:**

> "Micro-frontends extend the microservices concept to the UI: the frontend is composed of independently developed and deployed pieces, each owned by a separate team. The core value is organizational, not technical — when you have 10+ teams all deploying the same monolithic frontend, they block each other constantly. Micro-frontends give each team the ability to ship on their own schedule.
>
> The main composition strategies are build-time integration (MFEs as npm packages, simple but doesn't give independent deployment), Module Federation (MFEs exposed as remote URLs loaded at runtime — the modern standard for true independence), server-side composition (edge or SSR assembles HTML from multiple services), and iframes (complete isolation at the cost of UX).
>
> Module Federation is the most sophisticated approach. A remote MFE exposes modules at a URL — like `https://products.cdn.com/remoteEntry.js`. The shell imports them as dynamic imports at runtime. React and other shared dependencies can be configured as singletons to avoid being downloaded multiple times. Each MFE has its own webpack build, its own deployment pipeline, and can be updated without rebuilding the shell.
>
> The hard problems are design consistency (need a shared design system distributed via npm or Federation), routing (shell owns top-level routes, each MFE owns its sub-routes), cross-app communication (event bus pattern works well), and authentication (usually shared via cookie or Shell-provided context).
>
> The tradeoff that catches teams off guard: micro-frontends solve coordination problems but introduce technical complexity. If you have fewer than 5 teams or sections that communicate heavily, the overhead — multiple repos, multiple CI/CD pipelines, shared infrastructure like the design system — exceeds the benefit. I'd recommend feature-based architecture in a monorepo as the first scaling step, and only move to micro-frontends when team autonomy genuinely requires independent deployment."

---

## 18. Exercises

### Exercise 1 — Design the communication protocol

Design the complete communication protocol for a micro-frontend architecture with these MFEs:

- Shell (nav, auth badge, cart badge)
- Auth MFE (login/logout)
- Products MFE (browse and search)
- Cart MFE (cart management)
- Checkout MFE (payment flow)

Define: what events each MFE publishes, what it subscribes to, and what shared context the shell provides.

<details>
<summary>Solution</summary>

```typescript
// Shared EventMap type:
type EventMap = {
  // Auth events
  "auth:login-success": { user: User; token: string };
  "auth:logout": void;
  "auth:token-refreshed": { token: string };

  // Cart events
  "cart:item-added": { productId: string; quantity: number };
  "cart:item-removed": { productId: string };
  "cart:updated": { count: number; total: number; items: CartItem[] };
  "cart:cleared": void;

  // Checkout events
  "checkout:started": { cartSnapshot: CartItem[] };
  "checkout:completed": { orderId: string; total: number };
  "checkout:abandoned": void;

  // Navigation events
  "nav:go-to-cart": void;
  "nav:go-to-checkout": void;

  // Error events
  "app:error": {
    message: string;
    severity: "error" | "warning";
    retry?: () => void;
  };
};

// Shell:
// Provides via context: { theme, locale, user, token }
// Subscribes: auth:login-success → update user badge
//             auth:logout → clear user badge
//             cart:updated → update cart badge count
//             app:error → show toast
//             nav:go-to-cart → navigate to /cart
//             nav:go-to-checkout → navigate to /checkout

// Auth MFE:
// Publishes: auth:login-success, auth:logout, auth:token-refreshed
// Subscribes: nothing (self-contained)
// Provides via shell context: receives nothing — initiates the auth flow

// Products MFE:
// Publishes: cart:item-added (when user clicks "Add to Cart")
//            nav:go-to-cart (when user wants to view cart after adding)
//            app:error (on fetch failure)
// Subscribes: auth:login-success (to show personalized content)
//             auth:logout (to clear personalized content)

// Cart MFE:
// Publishes: cart:updated (on any cart change)
//            cart:cleared
//            nav:go-to-checkout
// Subscribes: cart:item-added (from Products MFE)
//             auth:logout (to clear cart display)

// Checkout MFE:
// Publishes: checkout:started, checkout:completed, checkout:abandoned
//            app:error (on payment failure)
// Subscribes: cart:updated (to get latest cart state on load)
//             auth:login-success (to pre-fill address from user profile)
```

</details>

---

## 🔗 Related Topics

- [`system-design/01-large-scale-architecture.md`](./01-large-scale-architecture.md) — Architecture decisions that precede MFEs
- [`system-design/02-feature-based-structure.md`](./02-feature-based-structure.md) — Monorepo alternative to MFEs
- [`javascript-core/15-pub-sub-systems.md`](../javascript-core/15-pub-sub-systems.md) — Event bus implementation
- [`performance/08-bundle-optimization.md`](../performance/08-bundle-optimization.md) — Bundle strategy for MFEs
- [`browser-internals/10-ssr-csr-isr-streaming.md`](../browser-internals/10-ssr-csr-isr-streaming.md) — SSR for server-side composed MFEs

---

<div align="center">

**Next:** [`system-design/04-state-management-design.md`](./04-state-management-design.md) →

</div>
