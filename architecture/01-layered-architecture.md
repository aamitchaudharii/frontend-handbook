npm install# 01 — Layered Architecture

> **"Layered architecture is the oldest and most intuitive way to structure complex systems. Every layer knows about the layer below it and nothing about the layer above it. That one rule, consistently applied, prevents the entire class of 'my UI is tangled with my business logic' bugs."**

Layered architecture divides an application into horizontal layers — each with a clearly defined responsibility, each depending only on the layer beneath it. It's the structural pattern underlying clean architecture, hexagonal architecture, and most well-structured large applications. This document covers the three-layer model for frontend applications, the rules that make it work, common violations, and how to apply it to real React codebases.

---

## 📚 Table of Contents

1. [The Three-Layer Model](#1-the-three-layer-model)
2. [The Presentation Layer](#2-the-presentation-layer)
3. [The Domain Layer](#3-the-domain-layer)
4. [The Infrastructure Layer](#4-the-infrastructure-layer)
5. [Dependency Rules](#5-dependency-rules)
6. [Data Flow Through Layers](#6-data-flow-through-layers)
7. [The Layer Boundary in Practice](#7-the-layer-boundary-in-practice)
8. [Ports and Adapters (Hexagonal Architecture)](#8-ports-and-adapters-hexagonal-architecture)
9. [Applying Layers in React](#9-applying-layers-in-react)
10. [Testing with Layers](#10-testing-with-layers)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. The Three-Layer Model

```
┌──────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                           │
│  React components, views, styles                             │
│  "What the user sees and interacts with"                     │
│                                                               │
│  Knows about: UI state, user events, rendering               │
│  Does NOT know: where data comes from, business rules        │
└──────────────────────────┬───────────────────────────────────┘
                           │ calls
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  DOMAIN LAYER                                                 │
│  Business logic, domain models, use cases                    │
│  "What the application does"                                  │
│                                                               │
│  Knows about: business concepts (User, Order, Cart)          │
│  Does NOT know: React, DOM, HTTP, database                   │
└──────────────────────────┬───────────────────────────────────┘
                           │ calls
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE LAYER                                         │
│  HTTP clients, local storage, WebSockets, analytics          │
│  "How the application talks to the outside world"            │
│                                                               │
│  Knows about: network, storage, external APIs                │
│  Does NOT know: business concepts or UI concerns             │
└──────────────────────────────────────────────────────────────┘
```

### Why Three Layers?

Each layer has a distinct reason to change:

```
Presentation changes when: UI design changes, UX requirements change, framework updates
Domain changes when:       Business rules change, new features, policy changes
Infrastructure changes when: API changes, new storage strategy, different analytics provider

By separating them:
  → Changing the UI doesn't affect business logic
  → Changing the HTTP client doesn't affect how orders are validated
  → A/B testing UI variants doesn't require changing any business code
```

---

## 2. The Presentation Layer

The presentation layer's job is exactly two things: display data to the user and translate user input into calls to the domain layer.

### What Belongs Here

```typescript
// ✅ PRESENTATION layer contents:

// Components that render UI
function OrderSummary({ orderId }: { orderId: string }) { ... }

// UI-only state (open/closed, hover, animation)
const [isExpanded, setIsExpanded] = useState(false);

// Event handlers that call domain
const handleSubmit = async (formData) => {
  await domain.checkout.placeOrder(formData); // ← calls domain
};

// Presentation-layer hooks (compose domain + UI state)
function useOrderSummaryViewModel(orderId: string) {
  const order = useOrder(orderId);          // domain data
  const [expanded, setExpanded] = useState(false); // UI state
  return { order, expanded, toggleExpand: () => setExpanded(e => !e) };
}
```

### What Doesn't Belong Here

```typescript
// ❌ Business logic in a component — violates presentation layer purity
function OrderCard({ order }: { order: Order }) {
  // Business rule: orders > $500 get free shipping — this is domain logic!
  const freeShipping = order.total > 500;
  const discount      = order.items.length > 10 ? order.total * 0.1 : 0;

  return (
    <div>
      {freeShipping && <Badge>Free Shipping</Badge>}
      <p>Total: ${order.total - discount}</p>
    </div>
  );
}

// ✅ Business rules in the domain layer, component is pure rendering
function OrderCard({ order }: { order: Order }) {
  // These are computed by the domain layer
  const { freeShipping, discountedTotal } = useOrderViewModel(order);
  return (
    <div>
      {freeShipping && <Badge>Free Shipping</Badge>}
      <p>Total: ${discountedTotal}</p>
    </div>
  );
}
```

---

## 3. The Domain Layer

The domain layer contains pure business logic — functions that transform domain objects, enforce business rules, and model the core concepts of the application.

### Characteristics of the Domain Layer

```typescript
// ✅ DOMAIN layer: pure functions, no framework, no I/O

// src/domain/order/orderDomain.ts

// Domain types
interface Order {
  id: string;
  items: OrderItem[];
  subtotal: number;
  shipping: number;
  promoCode?: string;
}

// Business rules as pure functions
function calculateShipping(order: Order): number {
  if (order.subtotal >= 500) return 0; // free above $500
  if (order.subtotal >= 100) return 5.99; // flat fee above $100
  return 12.99; // standard rate
}

function applyPromoCode(order: Order, code: string): Order {
  const discount = VALID_PROMO_CODES[code];
  if (!discount) throw new DomainError(`Invalid promo code: ${code}`);

  return {
    ...order,
    promoCode: code,
    subtotal: order.subtotal * (1 - discount),
  };
}

function validateOrder(order: Order): string[] {
  const errors: string[] = [];
  if (order.items.length === 0)
    errors.push("Order must contain at least one item");
  if (order.subtotal < 0) errors.push("Order total cannot be negative");
  return errors;
}

function getOrderTotal(order: Order): number {
  return order.subtotal + calculateShipping(order);
}

// Export as a domain module
export const orderDomain = {
  calculateShipping,
  applyPromoCode,
  validateOrder,
  getOrderTotal,
};
```

### Why No Framework Imports?

```typescript
// ❌ Domain layer importing React — wrong layer violation
import { useState } from "react";

export function useOrderValidation(order: Order) {
  const [errors, setErrors] = useState<string[]>([]);
  // ...
}
// Problem: this domain logic is now tied to React.
// Can't use it in a Node.js backend, a CLI, or a different framework.

// ✅ Domain logic as pure function — usable anywhere
export function validateOrder(order: Order): string[] {
  const errors: string[] = [];
  if (order.items.length === 0) errors.push("Order requires items");
  return errors;
}

// The HOOK lives in the presentation layer, calls the domain function:
// presentation/hooks/useOrderValidation.ts
function useOrderValidation(order: Order) {
  return useMemo(() => orderDomain.validateOrder(order), [order]);
}
```

---

## 4. The Infrastructure Layer

The infrastructure layer handles all communication with systems outside the application: HTTP APIs, browser storage, WebSockets, analytics, third-party SDKs.

### What Belongs Here

```typescript
// ✅ INFRASTRUCTURE layer: I/O, external communication

// src/infrastructure/http/apiClient.ts
export const apiClient = axios.create({ baseURL: config.apiUrl });

// src/infrastructure/orders/ordersApi.ts
export const ordersApi = {
  getOrder: (id: string) =>
    apiClient.get<Order>(`/orders/${id}`).then((r) => r.data),
  placeOrder: (data: PlaceOrderDto) =>
    apiClient.post<Order>("/orders", data).then((r) => r.data),
  cancelOrder: (id: string) => apiClient.delete(`/orders/${id}`),
};

// src/infrastructure/storage/localStorageAdapter.ts
export const cartStorage = {
  load: (): CartItem[] => JSON.parse(localStorage.getItem("cart") ?? "[]"),
  save: (items: CartItem[]) =>
    localStorage.setItem("cart", JSON.stringify(items)),
  clear: () => localStorage.removeItem("cart"),
};

// src/infrastructure/analytics/analyticsService.ts
export const analytics = {
  track: (event: string, props?: object) => mixpanel.track(event, props),
  identify: (userId: string, traits?: object) => mixpanel.identify(userId),
  page: (name: string) => mixpanel.track("Page Viewed", { page: name }),
};
```

### Infrastructure Doesn't Know Business Concepts

```typescript
// ❌ Infrastructure layer mixing business rules — wrong
// src/infrastructure/orders/ordersApi.ts
export async function placeOrderWithFreeShipping(order: Order) {
  // Business rule "free shipping" doesn't belong in the HTTP layer!
  if (order.total >= 500) {
    order.shipping = 0;
  }
  return apiClient.post("/orders", order);
}

// ✅ Clean separation: infrastructure just sends data
export const ordersApi = {
  placeOrder: (data: PlaceOrderDto) =>
    apiClient.post<Order>("/orders", data).then((r) => r.data),
  // No business logic here
};

// Domain layer applies the shipping rule before calling infrastructure
// Domain: const shippingCost = orderDomain.calculateShipping(cart);
// Presentation: calls domain, then passes result to infrastructure
```

---

## 5. Dependency Rules

The golden rule of layered architecture: **dependencies point downward only**.

```
ALLOWED:
  Presentation → Domain
  Presentation → Infrastructure (for things that don't need domain logic)
  Domain → (nothing above Domain)
  Infrastructure → (nothing above Infrastructure)

FORBIDDEN:
  Domain → Presentation       (domain must not import React, components, hooks)
  Domain → Infrastructure     (domain must not make HTTP calls, access storage)
  Infrastructure → Domain     (infrastructure must not know business concepts)
  Infrastructure → Presentation (infrastructure must not know about UI)
```

### Enforcing with TypeScript Path Aliases

```json
// tsconfig.json — paths make layer violations obvious
{
  "compilerOptions": {
    "paths": {
      "@/presentation/*": ["src/presentation/*"],
      "@/domain/*": ["src/domain/*"],
      "@/infrastructure/*": ["src/infrastructure/*"]
    }
  }
}
```

```javascript
// .eslintrc.js — enforce direction
module.exports = {
  rules: {
    "no-restricted-imports": [
      "error",
      {
        patterns: [
          {
            group: ["@/presentation/*"],
            importNames: [],
            message:
              "Domain and infrastructure must not import from presentation layer",
          },
        ],
      },
    ],
  },
  overrides: [
    {
      files: ["src/domain/**"],
      rules: {
        "no-restricted-imports": [
          "error",
          {
            patterns: [
              {
                group: ["@/presentation/*"],
                message: "Domain must not import from presentation",
              },
              {
                group: ["@/infrastructure/*"],
                message: "Domain must not import from infrastructure",
              },
              {
                group: ["react", "react-dom"],
                message: "Domain must not import from React",
              },
            ],
          },
        ],
      },
    },
  ],
};
```

---

## 6. Data Flow Through Layers

Understanding how data flows clarifies the responsibilities:

```
USER INTERACTION FLOW (top-down):

  User clicks "Place Order" button
        │
        ▼ [PRESENTATION]
  Component handles onClick
  Calls: placeOrderUseCase(cartItems, shippingAddress)
        │
        ▼ [DOMAIN]
  placeOrderUseCase:
    1. validateOrder(items) — apply business rules
    2. calculateShipping(items, address) — business logic
    3. buildOrderDto(items, address, shipping) — map to DTO
    4. calls ordersApi.placeOrder(dto) — calls infrastructure
        │
        ▼ [INFRASTRUCTURE]
  ordersApi.placeOrder:
    1. POST /api/orders
    2. Parse response
    3. Return Order object
        │
        ▼ [DOMAIN] (back up)
  placeOrderUseCase:
    5. Validates response
    6. Fires orderPlaced domain event
    7. Returns Order to caller
        │
        ▼ [PRESENTATION] (back up)
  Component receives Order
  Navigates to confirmation page
  Shows success notification
```

### Data Transformation at Layer Boundaries

```typescript
// INFRASTRUCTURE returns raw API data (server's format)
type ApiOrderResponse = {
  order_id: string;
  order_total: number;
  created_at: string; // ISO string
  line_items: ApiLineItem[];
};

// DOMAIN works with domain objects (application's format)
type Order = {
  id: string;
  total: number;
  createdAt: Date; // proper Date object
  items: OrderItem[];
};

// ADAPTER: at the infrastructure/domain boundary
// src/infrastructure/orders/orderAdapter.ts
function adaptApiOrder(api: ApiOrderResponse): Order {
  return {
    id: api.order_id,
    total: api.order_total,
    createdAt: new Date(api.created_at),
    items: api.line_items.map(adaptApiLineItem),
  };
}

export const ordersApi = {
  getOrder: async (id: string): Promise<Order> => {
    const raw = await apiClient.get<ApiOrderResponse>(`/orders/${id}`);
    return adaptApiOrder(raw.data); // transform at the boundary
  },
};
```

---

## 7. The Layer Boundary in Practice

### Feature Structure with Layers

```
src/features/orders/
  presentation/
    OrderList.tsx          ← renders order list UI
    OrderDetail.tsx        ← renders single order UI
    OrderSummaryCard.tsx   ← renders order card
    hooks/
      useOrders.ts         ← composes domain + TanStack Query
      useOrderViewModel.ts ← transforms domain data for UI

  domain/
    orderDomain.ts         ← pure business logic functions
    useOrderUseCase.ts     ← orchestrates domain operations
    types.ts               ← Order, OrderItem domain types

  infrastructure/
    ordersApi.ts           ← HTTP calls
    orderAdapter.ts        ← API response → domain object
    orderStorage.ts        ← local persistence
```

### The Use Case — Orchestrating Domain Operations

```typescript
// domain/useOrderUseCase.ts
// A use case coordinates domain functions and infrastructure calls
// It's not quite presentation (no React) and not quite pure domain (calls infrastructure)
// Some architectures put this in domain, some create a separate "application" layer

import { orderDomain } from "./orderDomain";
import { ordersApi } from "../infrastructure/ordersApi";
import { eventBus } from "@/infrastructure/eventBus";

export const orderUseCases = {
  // Place order: orchestrates validation + shipping calc + API call + event
  async placeOrder(cart: Cart, address: Address): Promise<Order> {
    // Domain validation
    const errors = orderDomain.validateCart(cart);
    if (errors.length > 0) throw new DomainError(errors.join(", "));

    // Domain business logic
    const shipping = orderDomain.calculateShipping(cart, address);
    const dto = orderDomain.buildPlaceOrderDto(cart, address, shipping);

    // Infrastructure call
    const order = await ordersApi.placeOrder(dto);

    // Side effects via events (don't call analytics/notifications directly)
    eventBus.publish("order:placed", {
      orderId: order.id,
      total: order.total,
      items: cart.items.length,
    });

    return order;
  },

  // Cancel order: domain rule check + API call
  async cancelOrder(orderId: string, reason: string): Promise<void> {
    const order = await ordersApi.getOrder(orderId);

    if (!orderDomain.canCancel(order)) {
      throw new DomainError("Order cannot be cancelled at this stage");
    }

    await ordersApi.cancelOrder(orderId, reason);
    eventBus.publish("order:cancelled", { orderId, reason });
  },
};
```

---

## 8. Ports and Adapters (Hexagonal Architecture)

Hexagonal architecture (Ports & Adapters) is a refinement of layered architecture that makes the domain completely independent of infrastructure through interfaces.

```
                    ┌────────────────────────────┐
   UI Adapter  ───► │                            │ ◄─── REST Adapter
   (React)          │    DOMAIN / APPLICATION    │      (fetch)
                    │                            │
   Test Adapter ───►│  Pure business logic only  │ ◄─── Storage Adapter
   (mocks)          │  No framework, no I/O      │      (localStorage)
                    │                            │
                    └────────────────────────────┘
                         ↑ depends on interfaces (ports)
                         ↑ adapters implement those interfaces
```

### Port Definition (Interface in Domain)

```typescript
// domain/ports/OrderRepository.ts
// A "port" is an interface the domain defines
// Infrastructure provides the implementation ("adapter")

export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  findByUserId(userId: string): Promise<Order[]>;
  save(order: Order): Promise<Order>;
  delete(id: string): Promise<void>;
}
```

### Adapter Implementation (Infrastructure)

```typescript
// infrastructure/orders/HttpOrderRepository.ts
// "Adapter": implements the port interface using HTTP
import type { OrderRepository } from "@/domain/ports/OrderRepository";

export class HttpOrderRepository implements OrderRepository {
  async findById(id: string): Promise<Order | null> {
    try {
      const raw = await apiClient.get<ApiOrderResponse>(`/orders/${id}`);
      return adaptApiOrder(raw.data);
    } catch (err) {
      if (isNotFoundError(err)) return null;
      throw err;
    }
  }

  async findByUserId(userId: string): Promise<Order[]> {
    const raw = await apiClient.get<ApiOrderResponse[]>(
      `/users/${userId}/orders`,
    );
    return raw.data.map(adaptApiOrder);
  }

  async save(order: Order): Promise<Order> {
    const raw = await apiClient.post<ApiOrderResponse>("/orders", order);
    return adaptApiOrder(raw.data);
  }

  async delete(id: string): Promise<void> {
    await apiClient.delete(`/orders/${id}`);
  }
}

// Test adapter (in-memory implementation for tests):
export class InMemoryOrderRepository implements OrderRepository {
  #orders = new Map<string, Order>();

  async findById(id: string): Promise<Order | null> {
    return this.#orders.get(id) ?? null;
  }
  async save(order: Order): Promise<Order> {
    this.#orders.set(order.id, order);
    return order;
  }
  // ...
}
```

### Domain Uses the Port Interface

```typescript
// domain/useOrderUseCase.ts — depends on the interface, not the implementation
export function createOrderUseCases(orderRepo: OrderRepository) {
  return {
    async getOrderHistory(userId: string): Promise<Order[]> {
      const orders = await orderRepo.findByUserId(userId);
      return orders.sort(
        (a, b) => b.createdAt.getTime() - a.createdAt.getTime(),
      );
    },
  };
}

// Wired together at app startup (dependency injection):
const orderUseCases = createOrderUseCases(new HttpOrderRepository());

// In tests: inject InMemoryOrderRepository instead
const orderUseCases = createOrderUseCases(new InMemoryOrderRepository());
```

---

## 9. Applying Layers in React

### Presentation Layer Hooks

```typescript
// presentation/hooks/useOrders.ts
// Bridges domain + TanStack Query + UI concerns

import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { orderUseCases } from "@/domain/orderUseCases";

// READ: domain data for presentation
export function useOrders(userId: string) {
  return useQuery({
    queryKey: ["orders", userId],
    queryFn: () => orderUseCases.getOrderHistory(userId),
  });
}

// WRITE: mutation that calls domain use case
export function usePlaceOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ cart, address }: PlaceOrderArgs) =>
      orderUseCases.placeOrder(cart, address),

    onSuccess: (order) => {
      // Presentation-layer concern: update UI state
      queryClient.invalidateQueries({ queryKey: ["orders"] });
    },
  });
}
```

### View Model Pattern

```typescript
// presentation/hooks/useOrderViewModel.ts
// Transforms domain objects into view-friendly shapes

interface OrderViewModel {
  id: string;
  formattedDate: string;
  formattedTotal: string;
  statusLabel: string;
  statusColor: "green" | "yellow" | "red" | "gray";
  canCancel: boolean;
  itemCount: number;
}

function toOrderViewModel(order: Order): OrderViewModel {
  return {
    id: order.id,
    formattedDate: formatDate(order.createdAt, "MMM d, yyyy"),
    formattedTotal: formatCurrency(order.total, order.currency),
    statusLabel: ORDER_STATUS_LABELS[order.status],
    statusColor: ORDER_STATUS_COLORS[order.status],
    canCancel: orderDomain.canCancel(order),
    itemCount: order.items.reduce((sum, i) => sum + i.quantity, 0),
  };
}

export function useOrderViewModel(orderId: string) {
  const { data: order, ...rest } = useQuery({
    queryKey: ["orders", orderId],
    queryFn: () => ordersApi.getOrder(orderId),
  });

  return {
    ...rest,
    viewModel: order ? toOrderViewModel(order) : null,
  };
}
```

---

## 10. Testing with Layers

Layered architecture makes each layer independently testable:

```typescript
// DOMAIN LAYER TESTS: pure unit tests — no mocks, no framework
describe('orderDomain', () => {
  test('calculateShipping returns 0 for orders over $500', () => {
    const order = buildOrder({ subtotal: 600 });
    expect(orderDomain.calculateShipping(order)).toBe(0);
  });

  test('validateOrder catches empty item list', () => {
    const errors = orderDomain.validateOrder({ items: [] });
    expect(errors).toContain('Order must contain at least one item');
  });
});
// No React, no HTTP, no setup — milliseconds to run

// INFRASTRUCTURE LAYER TESTS: HTTP interactions
// Use MSW to mock the server
describe('ordersApi', () => {
  test('getOrder transforms response correctly', async () => {
    server.use(http.get('/api/orders/42', () =>
      HttpResponse.json({ order_id: '42', order_total: 29.99, created_at: '2024-01-15T00:00:00Z', line_items: [] })
    ));

    const order = await ordersApi.getOrder('42');
    expect(order.id).toBe('42');
    expect(order.total).toBe(29.99);
    expect(order.createdAt).toBeInstanceOf(Date);
  });
});

// PRESENTATION LAYER TESTS: component behavior
// Domain and infrastructure are mocked
describe('OrderCard', () => {
  test('shows "Free Shipping" badge for eligible orders', () => {
    // Inject mock data through TanStack Query
    render(<OrderCard orderId="42" />, { wrapper: createQueryWrapper({
      ['orders', '42']: buildOrder({ subtotal: 600 }),
    })});

    expect(screen.getByText('Free Shipping')).toBeInTheDocument();
  });
});
```

---

## 11. Good Practices

### ✅ One-way dependency enforcement with ESLint

```javascript
// Catch violations at lint time, not review time
'no-restricted-imports': ['error', {
  patterns: [
    { group: ['react', 'react-dom'], message: 'Domain layer must not import React' }
  ]
}]
// Applied in src/domain/**
```

### ✅ Domain types are the source of truth

```typescript
// Domain types drive everything
// Infrastructure adapts to them (not vice versa)
// Presentation renders them (not vice versa)
```

### ✅ Use cases orchestrate, don't do work directly

```typescript
// ✅ Use case: thin orchestration layer
async function placeOrder(cart: Cart, address: Address) {
  orderDomain.validate(cart); // delegates to domain
  const shipping = orderDomain.calculateShipping(cart, address); // delegates
  const dto = orderDomain.buildDto(cart, address, shipping); // delegates
  const order = await ordersApi.place(dto); // delegates to infrastructure
  eventBus.publish("order:placed", order); // delegates to event bus
  return order;
}
```

---

## 12. Bad Practices

### ❌ Fetching data in domain logic

```typescript
// ❌ Domain calls HTTP — domain should be pure
function validateOrder(orderId: string) {
  const order = await fetch(`/api/orders/${orderId}`).then((r) => r.json()); // ← I/O in domain!
  if (order.items.length === 0) throw new Error("Empty order");
}

// ✅ Data is passed to domain, fetched elsewhere
function validateOrder(order: Order) {
  if (order.items.length === 0) throw new DomainError("Empty order");
}
```

### ❌ UI logic in the domain

```typescript
// ❌ Domain formatting display values
function formatOrderForDisplay(order: Order): string {
  return `Order #${order.id} — $${order.total.toFixed(2)} (${order.status})`; // display concern
}

// ✅ Presentation layer handles display
const OrderTitle = ({ order }: { order: Order }) =>
  `Order #${order.id} — ${formatCurrency(order.total)} (${STATUS_LABELS[order.status]})`;
```

---

## 13. Common Mistakes

### Mistake 1 — "Thin domain" syndrome

```typescript
// ❌ Domain layer that does nothing useful
// domain/orderDomain.ts
export const orderDomain = {
  getTotal: (order: Order) => order.total, // just a getter!
  getItems: (order: Order) => order.items, // just a getter!
};
// The real business logic ended up in components — domain is just a wrapper

// ✅ Domain layer with real logic
export const orderDomain = {
  calculateTotal: (order: Order) =>
    order.subtotal + calculateShipping(order) - getDiscount(order),
  canAddItem: (order: Order, item: Product) =>
    order.items.length < MAX_ITEMS && item.inStock,
  applyPromoCode: (order: Order, code: string) => {
    /* validation + discount */
  },
  isFreeShipping: (order: Order) => order.subtotal >= FREE_SHIPPING_THRESHOLD,
};
```

### Mistake 2 — Wrong layer for "smart" hooks

```typescript
// ❌ Hook mixes domain logic + infrastructure + presentation — no layer
function useOrderWithFreeShipping(orderId: string) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(`/api/orders/${orderId}`) // infrastructure
      .then((r) => r.json())
      .then((order) => {
        const shipping = order.total >= 500 ? 0 : 12.99; // domain logic!
        setData({ ...order, shipping }); // presentation state
      });
  }, [orderId]);

  return data;
}

// ✅ Separate concerns into correct layers
function useOrder(orderId: string) {
  return useQuery({
    queryKey: ["order", orderId],
    queryFn: () => ordersApi.getOrder(orderId), // infrastructure
    select: (order) => ({
      // domain + view model
      ...order,
      shipping: orderDomain.calculateShipping(order),
    }),
  });
}
```

---

## 14. Interview-Level Explanation

> **"How do you structure a frontend application using layered architecture? Why does it matter?"**

**Strong answer:**

> "Layered architecture divides the codebase into three horizontal layers: presentation, domain, and infrastructure. Each layer has a single responsibility and dependencies only flow downward — presentation can call domain, domain can call infrastructure, but never the reverse.
>
> The presentation layer is React components, hooks that connect components to data, and UI-specific state. Its job is purely: show this data, respond to this event. It knows nothing about where data comes from or what business rules apply.
>
> The domain layer is pure business logic — functions that transform domain objects, enforce business rules, and model the core concepts. Order validation, shipping calculation, discount application. Critically, the domain layer imports nothing from React, nothing from the HTTP layer, nothing from storage. It's pure TypeScript. This makes it trivially testable: you call the function with inputs and assert on the output. No mocking, no setup, no async. Just data in, data out.
>
> The infrastructure layer handles all communication with the outside world: HTTP clients, localStorage, WebSockets, analytics. It transforms server responses into domain objects at the boundary. The domain never sees raw API shapes — the adapter converts them.
>
> Why it matters: when a component contains both `fetch()` calls and discount calculation logic, you can't test the business logic without setting up network mocking. You can't change the API format without touching business logic. You can't reuse the business logic in a different context. Separating layers solves all three: the business logic is a pure function you test directly, the API adapter is the only thing that knows the server's shape, and the component is the only thing that knows about React.
>
> The most common violation is the 'smart component anti-pattern': a 400-line React component that fetches data, applies business rules, manages complex state, and renders UI — all mixed together. Splitting that into presentation (renders UI), domain (applies rules), and infrastructure (fetches data) makes each part independently maintainable and testable."

---

## 15. Exercises

### Exercise 1 — Layer classification

For each piece of code, identify which layer it belongs to and why:

```typescript
// a)
const isEligibleForLoyaltyPoints = (order: Order): boolean =>
  order.total >= 50 && order.status === 'completed';

// b)
function OrderStatusBadge({ status }: { status: OrderStatus }) {
  const color = { pending: 'yellow', completed: 'green', cancelled: 'red' }[status];
  return <span className={`badge badge--${color}`}>{status}</span>;
}

// c)
export const ordersApi = {
  getOrder: (id: string) => apiClient.get<ApiOrderResponse>(`/orders/${id}`).then(r => r.data),
};

// d)
function calculateOrderDiscount(order: Order, customer: Customer): number {
  if (customer.tier === 'gold') return order.subtotal * 0.15;
  if (customer.tier === 'silver') return order.subtotal * 0.10;
  return 0;
}

// e)
function useOrderHistory(userId: string) {
  return useQuery({
    queryKey: ['orders', userId],
    queryFn:  () => ordersApi.getOrdersByUser(userId),
  });
}
```

<details>
<summary>Answers</summary>

```
a) DOMAIN — pure function, no I/O, no framework, expresses a business rule
   (eligibility for loyalty points is a business concept)

b) PRESENTATION — React component, rendering only, no business logic
   (color mapping for status is a display concern)

c) INFRASTRUCTURE — HTTP call, returns raw data from API
   (API communication belongs in infrastructure)

d) DOMAIN — pure function, business rules (discount tiers), no I/O
   (discount calculation is core business logic)

e) PRESENTATION — React hook using TanStack Query
   (bridges domain data to React component lifecycle)
   Note: it calls infrastructure directly (ordersApi) — some architectures
   would put this call through a domain use case instead.
   Acceptable for simple reads; use a use case for writes/complex flows.
```

</details>

---

## 🔗 Related Topics

- [`system-design/01-large-scale-architecture.md`](../system-design/01-large-scale-architecture.md) — How layers fit in large app architecture
- [`system-design/02-feature-based-structure.md`](../system-design/02-feature-based-structure.md) — Layers within feature modules
- [`testing/01-unit-testing.md`](../testing/01-unit-testing.md) — Testing pure domain functions
- [`architecture/02-clean-architecture.md`](./02-clean-architecture.md) — Clean Architecture extends layered architecture

---

<div align="center">

**Next:** [`architecture/02-clean-architecture.md`](./02-clean-architecture.md) →

</div>
