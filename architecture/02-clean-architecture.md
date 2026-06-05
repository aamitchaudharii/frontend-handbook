# 02 — Clean Architecture

> **"The goal of Clean Architecture is simple: separate the things that change for different reasons. When the database changes, the business rules shouldn't. When the UI framework changes, the business rules shouldn't. When the business rules change, the UI and database shouldn't need to — unless the rules drive the UI or database schema."**
> — Robert C. Martin (paraphrased)

Clean Architecture is a refinement of layered architecture by Robert C. Martin ("Uncle Bob"). It adds two critical ideas to basic layering: the **Dependency Inversion Principle** (high-level modules should not depend on low-level modules — both should depend on abstractions) and the **Stable Abstractions Principle** (the more stable a component is, the more abstract it should be). Applied to the frontend, Clean Architecture produces a codebase where the business logic is completely framework-agnostic and testable in isolation from everything else.

---

## 📚 Table of Contents

1. [The Dependency Rule](#1-the-dependency-rule)
2. [The Concentric Ring Model](#2-the-concentric-ring-model)
3. [Entities — Core Business Objects](#3-entities--core-business-objects)
4. [Use Cases — Application Business Rules](#4-use-cases--application-business-rules)
5. [Interface Adapters](#5-interface-adapters)
6. [Frameworks and Drivers](#6-frameworks-and-drivers)
7. [Dependency Inversion in Practice](#7-dependency-inversion-in-practice)
8. [Clean Architecture in a React Application](#8-clean-architecture-in-a-react-application)
9. [The Humble Object Pattern](#9-the-humble-object-pattern)
10. [Screaming Architecture](#10-screaming-architecture)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. The Dependency Rule

Clean Architecture has one fundamental rule:

> **Source code dependencies must point only inward — toward higher-level policies.**

```
                    ┌───────────────────┐
                    │  Frameworks &     │
                    │  Drivers          │ ← outermost: most volatile
                    │  ┌─────────────┐  │
                    │  │  Interface  │  │
                    │  │  Adapters   │  │
                    │  │  ┌───────┐  │  │
                    │  │  │ Use   │  │  │
                    │  │  │ Cases │  │  │
                    │  │  │ ┌───┐ │  │  │
                    │  │  │ │Ent│ │  │  │
                    │  │  │ │ity│ │  │  │ ← innermost: most stable
                    │  │  │ └───┘ │  │  │
                    │  │  └───────┘  │  │
                    │  └─────────────┘  │
                    └───────────────────┘
      Dependencies → → → → → → → → inward only
      Outer layers know about inner layers
      Inner layers know NOTHING about outer layers
```

### What the Dependency Rule Means Practically

```typescript
// ✅ ALLOWED: Use case imports from entities (inner → more inner)
import { Order } from "@/entities/Order";
const total = order.calculateTotal();

// ✅ ALLOWED: Adapter imports from use case (outer → inner)
import { placeOrderUseCase } from "@/use-cases/placeOrder";
await placeOrderUseCase.execute(cart, address);

// ❌ FORBIDDEN: Entity imports from use case (inner knows about outer)
// src/entities/Order.ts
import { placeOrderUseCase } from "@/use-cases/placeOrder"; // ← violates rule!

// ❌ FORBIDDEN: Use case imports from React/Redux
// src/use-cases/placeOrder.ts
import { useDispatch } from "react-redux"; // ← violates rule!
import { store } from "@/store/store"; // ← violates rule!

// ❌ FORBIDDEN: Use case directly calls fetch
// src/use-cases/placeOrder.ts
const response = await fetch("/api/orders"); // ← violates rule!
```

---

## 2. The Concentric Ring Model

```
RING 1 — ENTITIES (innermost)
  Pure domain objects.
  Enterprise business rules.
  Change only when fundamental business rules change.
  No dependencies on anything outside this ring.

RING 2 — USE CASES
  Application-specific business rules.
  Orchestrate the flow of data to and from entities.
  Change when application requirements change.
  Depend only on entities.
  Know nothing about UI, database, or framework.

RING 3 — INTERFACE ADAPTERS
  Translate between use cases and external systems.
  Presenters, controllers, repositories.
  Convert data formats at the boundary.
  Change when external systems change (API format, framework, database).

RING 4 — FRAMEWORKS & DRIVERS (outermost)
  React, Redux, Express, Firebase, localStorage.
  The most volatile, most likely to change.
  Written to satisfy inner rings.
  Thin as possible — just plumbing.
```

---

## 3. Entities — Core Business Objects

Entities encapsulate enterprise-wide business rules. They are the most stable objects in the system.

```typescript
// src/entities/Order.ts
// An entity encapsulates both data and the invariants/rules that govern it

export class Order {
  readonly id: string;
  readonly customerId: string;
  readonly items: readonly OrderItem[];
  readonly status: OrderStatus;
  readonly createdAt: Date;

  constructor(props: OrderProps) {
    // Enforce invariants at construction time
    if (props.items.length === 0) {
      throw new EntityError("Order must have at least one item");
    }
    if (props.items.some((i) => i.quantity <= 0)) {
      throw new EntityError("Item quantities must be positive");
    }

    this.id = props.id ?? generateId();
    this.customerId = props.customerId;
    this.items = Object.freeze([...props.items]);
    this.status = props.status ?? "pending";
    this.createdAt = props.createdAt ?? new Date();
  }

  // Business methods — entity behavior
  get subtotal(): number {
    return this.items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0,
    );
  }

  get itemCount(): number {
    return this.items.reduce((sum, item) => sum + item.quantity, 0);
  }

  canBeCancelled(): boolean {
    return this.status === "pending" || this.status === "processing";
  }

  canBeModified(): boolean {
    return this.status === "pending";
  }

  withStatus(newStatus: OrderStatus): Order {
    if (!this.isValidStatusTransition(newStatus)) {
      throw new EntityError(
        `Cannot transition order from ${this.status} to ${newStatus}`,
      );
    }
    return new Order({ ...this.toProps(), status: newStatus });
  }

  withAddedItem(item: OrderItem): Order {
    if (!this.canBeModified()) {
      throw new EntityError(`Cannot add items to ${this.status} order`);
    }
    const existing = this.items.find((i) => i.productId === item.productId);
    const updatedItems = existing
      ? this.items.map((i) =>
          i.productId === item.productId
            ? { ...i, quantity: i.quantity + item.quantity }
            : i,
        )
      : [...this.items, item];

    return new Order({ ...this.toProps(), items: updatedItems });
  }

  // Valid transitions: pending → processing → shipped → delivered
  //                   pending → cancelled, processing → cancelled
  private isValidStatusTransition(to: OrderStatus): boolean {
    const TRANSITIONS: Record<OrderStatus, OrderStatus[]> = {
      pending: ["processing", "cancelled"],
      processing: ["shipped", "cancelled"],
      shipped: ["delivered"],
      delivered: [],
      cancelled: [],
    };
    return TRANSITIONS[this.status]?.includes(to) ?? false;
  }

  toProps(): OrderProps {
    return {
      id: this.id,
      customerId: this.customerId,
      items: [...this.items],
      status: this.status,
      createdAt: this.createdAt,
    };
  }
}
```

### Value Objects (Simpler Entities)

```typescript
// Value objects: immutable, equality by value (not identity)
class Money {
  readonly amount: number;
  readonly currency: string;

  constructor(amount: number, currency: string) {
    if (amount < 0) throw new EntityError("Money amount cannot be negative");
    if (currency.length !== 3)
      throw new EntityError("Currency must be ISO 4217 code");
    this.amount = Math.round(amount * 100) / 100; // 2 decimal places
    this.currency = currency.toUpperCase();
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new EntityError(
        `Cannot add ${this.currency} and ${other.currency}`,
      );
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  toString(): string {
    return new Intl.NumberFormat("en-US", {
      style: "currency",
      currency: this.currency,
    }).format(this.amount);
  }
}
```

---

## 4. Use Cases — Application Business Rules

Use cases (also called "interactors") orchestrate domain operations to fulfill specific application requirements.

```typescript
// src/use-cases/placeOrder/PlaceOrderUseCase.ts

// Input: what the use case receives
export interface PlaceOrderInput {
  customerId: string;
  cartItems: CartItem[];
  shippingAddress: Address;
  paymentMethodId: string;
  promoCode?: string;
}

// Output: what the use case returns
export interface PlaceOrderOutput {
  order: Order;
  confirmationId: string;
  estimatedDelivery: Date;
}

// Ports: interfaces the use case depends on (implemented by adapters)
export interface OrderRepository {
  save(order: Order): Promise<Order>;
}
export interface PaymentGateway {
  charge(amount: Money, methodId: string): Promise<{ transactionId: string }>;
}
export interface NotificationService {
  sendOrderConfirmation(order: Order): Promise<void>;
}

// The use case itself
export class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly paymentGateway: PaymentGateway,
    private readonly notifications: NotificationService,
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderOutput> {
    // 1. Build the Order entity (entity validates invariants)
    const order = new Order({
      customerId: input.customerId,
      items: input.cartItems.map(cartItemToOrderItem),
      status: "pending",
    });

    // 2. Apply promo code if provided
    const finalOrder = input.promoCode
      ? this.applyPromoCode(order, input.promoCode)
      : order;

    // 3. Calculate total and charge payment
    const shipping = this.calculateShipping(finalOrder, input.shippingAddress);
    const total = new Money(finalOrder.subtotal + shipping, "USD");
    const { transactionId } = await this.paymentGateway.charge(
      total,
      input.paymentMethodId,
    );

    // 4. Transition order to processing
    const processedOrder = finalOrder.withStatus("processing");

    // 5. Persist
    const savedOrder = await this.orderRepo.save(processedOrder);

    // 6. Notify
    await this.notifications.sendOrderConfirmation(savedOrder);

    return {
      order: savedOrder,
      confirmationId: transactionId,
      estimatedDelivery: this.calculateDeliveryDate(input.shippingAddress),
    };
  }

  private applyPromoCode(order: Order, code: string): Order {
    const promo = PROMO_CODES[code];
    if (!promo || !promo.isValid())
      throw new UseCaseError(`Invalid promo code: ${code}`);
    // Apply discount via value object arithmetic
    const discount = new Money(order.subtotal * promo.discountRate, "USD");
    return new Order({
      ...order.toProps(),
      items: applyDiscountToItems(order.items, discount),
    });
  }

  private calculateShipping(order: Order, address: Address): number {
    if (order.subtotal >= 500) return 0;
    return address.country === "US" ? 9.99 : 24.99;
  }

  private calculateDeliveryDate(address: Address): Date {
    const days = address.country === "US" ? 5 : 14;
    const date = new Date();
    date.setDate(date.getDate() + days);
    return date;
  }
}
```

---

## 5. Interface Adapters

Interface adapters convert data between the format used by use cases and the format used by external systems (databases, HTTP, UI framework).

### Repository Adapter

```typescript
// src/adapters/repositories/HttpOrderRepository.ts
// Implements OrderRepository interface using HTTP

import type { OrderRepository } from "@/use-cases/placeOrder/PlaceOrderUseCase";
import { Order } from "@/entities/Order";

export class HttpOrderRepository implements OrderRepository {
  async save(order: Order): Promise<Order> {
    const dto = this.toApiDto(order);
    const response = await apiClient.post<ApiOrderResponse>("/api/orders", dto);
    return this.toDomain(response.data);
  }

  async findById(id: string): Promise<Order | null> {
    try {
      const response = await apiClient.get<ApiOrderResponse>(
        `/api/orders/${id}`,
      );
      return this.toDomain(response.data);
    } catch (err) {
      if (isNotFoundError(err)) return null;
      throw err;
    }
  }

  // Transform domain entity → API format
  private toApiDto(order: Order): ApiCreateOrderDto {
    return {
      customer_id: order.customerId,
      items: order.items.map((item) => ({
        product_id: item.productId,
        qty: item.quantity,
        unit_price: item.price,
      })),
    };
  }

  // Transform API response → domain entity
  private toDomain(api: ApiOrderResponse): Order {
    return new Order({
      id: api.order_id,
      customerId: api.customer_id,
      status: api.order_status as OrderStatus,
      createdAt: new Date(api.created_at),
      items: api.line_items.map((item) => ({
        productId: item.product_id,
        name: item.product_name,
        price: item.unit_price,
        quantity: item.qty,
      })),
    });
  }
}
```

### Presenter Adapter

```typescript
// src/adapters/presenters/OrderPresenter.ts
// Transforms domain objects → view model for the UI

import { Order } from "@/entities/Order";

export interface OrderViewModel {
  id: string;
  displayId: string; // "#ORD-001234"
  formattedTotal: string; // "$29.99"
  formattedDate: string; // "Jan 15, 2024"
  statusLabel: string; // "Processing"
  statusVariant: "success" | "warning" | "danger" | "default";
  canCancel: boolean;
  itemSummary: string; // "3 items"
}

export class OrderPresenter {
  present(order: Order): OrderViewModel {
    return {
      id: order.id,
      displayId: `#ORD-${order.id.slice(-6).toUpperCase()}`,
      formattedTotal: this.formatCurrency(order.subtotal),
      formattedDate: this.formatDate(order.createdAt),
      statusLabel: this.getStatusLabel(order.status),
      statusVariant: this.getStatusVariant(order.status),
      canCancel: order.canBeCancelled(),
      itemSummary: `${order.itemCount} ${order.itemCount === 1 ? "item" : "items"}`,
    };
  }

  presentList(orders: Order[]): OrderViewModel[] {
    return orders.map((o) => this.present(o));
  }

  private formatCurrency(amount: number): string {
    return new Intl.NumberFormat("en-US", {
      style: "currency",
      currency: "USD",
    }).format(amount);
  }

  private formatDate(date: Date): string {
    return date.toLocaleDateString("en-US", {
      year: "numeric",
      month: "short",
      day: "numeric",
    });
  }

  private getStatusLabel(status: OrderStatus): string {
    const labels: Record<OrderStatus, string> = {
      pending: "Pending",
      processing: "Processing",
      shipped: "Shipped",
      delivered: "Delivered",
      cancelled: "Cancelled",
    };
    return labels[status];
  }

  private getStatusVariant(
    status: OrderStatus,
  ): OrderViewModel["statusVariant"] {
    const variants: Record<OrderStatus, OrderViewModel["statusVariant"]> = {
      pending: "warning",
      processing: "warning",
      shipped: "default",
      delivered: "success",
      cancelled: "danger",
    };
    return variants[status];
  }
}
```

---

## 6. Frameworks and Drivers

The outermost ring is where React, the HTTP client, localStorage, and other external tools live. They are kept as thin as possible.

```typescript
// src/frameworks/react/OrdersPage.tsx
// Pure plumbing: wires React + TanStack Query → adapters → use cases

import React from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';
import { container } from '@/infrastructure/container'; // DI container

function OrdersPage() {
  // Wires framework (TanStack Query) to use case via adapter
  const { data: viewModels, isLoading } = useQuery({
    queryKey: ['orders'],
    queryFn:  () => container.getOrdersUseCase.execute({ userId: currentUser.id }),
    select:   (orders) => container.orderPresenter.presentList(orders),
  });

  const cancelMutation = useMutation({
    mutationFn: (orderId: string) =>
      container.cancelOrderUseCase.execute({ orderId }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['orders'] }),
  });

  if (isLoading) return <OrdersSkeleton />;

  return (
    <OrderList
      orders={viewModels ?? []}
      onCancel={(orderId) => cancelMutation.mutate(orderId)}
    />
  );
}
```

### Dependency Injection Container

```typescript
// src/infrastructure/container.ts
// Wire everything together at the boundary

import { PlaceOrderUseCase } from "@/use-cases/placeOrder/PlaceOrderUseCase";
import { GetOrdersUseCase } from "@/use-cases/getOrders/GetOrdersUseCase";
import { CancelOrderUseCase } from "@/use-cases/cancelOrder/CancelOrderUseCase";
import { HttpOrderRepository } from "@/adapters/repositories/HttpOrderRepository";
import { StripePaymentGateway } from "@/adapters/gateways/StripePaymentGateway";
import { EmailNotifications } from "@/adapters/gateways/EmailNotifications";
import { OrderPresenter } from "@/adapters/presenters/OrderPresenter";

// Create adapters (implementations)
const orderRepo = new HttpOrderRepository();
const paymentGateway = new StripePaymentGateway(config.stripeKey);
const notifications = new EmailNotifications(config.emailService);

// Create use cases (inject dependencies)
export const container = {
  placeOrderUseCase: new PlaceOrderUseCase(
    orderRepo,
    paymentGateway,
    notifications,
  ),
  getOrdersUseCase: new GetOrdersUseCase(orderRepo),
  cancelOrderUseCase: new CancelOrderUseCase(orderRepo, notifications),
  orderPresenter: new OrderPresenter(),
};

// Test container (swap adapters for test doubles)
export const testContainer = {
  placeOrderUseCase: new PlaceOrderUseCase(
    new InMemoryOrderRepository(), // test double
    new FakePaymentGateway(), // test double
    new NoOpNotifications(), // test double
  ),
  // ...
};
```

---

## 7. Dependency Inversion in Practice

The Dependency Inversion Principle (DIP) is the key mechanism that makes Clean Architecture work: use cases define interfaces they need; adapters implement those interfaces.

```
Without DIP:               With DIP (Inversion):
  UseCase                    UseCase
    ↓ imports directly         ↓ imports interface (port)
  HttpOrderRepo              IOrderRepository  ← abstract
                               ↑ implements
                             HttpOrderRepo     ← concrete
                             InMemoryOrderRepo  ← test double

The use case never changes when the repository implementation changes.
```

### The Interface Belongs to the Use Case

```typescript
// ✅ Interface defined IN the use case layer (inner ring)
// src/use-cases/placeOrder/PlaceOrderUseCase.ts
export interface IOrderRepository {
  save(order: Order): Promise<Order>;
  findById(id: string): Promise<Order | null>;
}

// Implementation in adapter layer (outer ring)
// src/adapters/repositories/HttpOrderRepository.ts
export class HttpOrderRepository implements IOrderRepository {
  // ...
}

// The use case imports the interface (inner → inner)
// The adapter imports the interface (outer → inner)
// They never need to know about each other

// ❌ WRONG: Interface defined in adapter layer
// src/adapters/repositories/IOrderRepository.ts  ← in the wrong ring!
export interface IOrderRepository { ... }
// This would make inner rings depend on the outer adapter layer
```

---

## 8. Clean Architecture in a React Application

### Directory Structure

```
src/
  entities/              ← Ring 1: pure domain objects
    Order.ts
    Cart.ts
    Customer.ts
    shared/
      Money.ts
      Address.ts
      EntityError.ts

  use-cases/             ← Ring 2: application business rules
    place-order/
      PlaceOrderUseCase.ts
      PlaceOrderUseCase.test.ts   ← unit test (no framework, no I/O)
      types.ts                    ← Input/Output interfaces
    get-orders/
    cancel-order/
    checkout/

  adapters/              ← Ring 3: interface adapters
    repositories/
      HttpOrderRepository.ts
      InMemoryOrderRepository.ts  ← test double
    gateways/
      StripePaymentGateway.ts
      FakePaymentGateway.ts       ← test double
    presenters/
      OrderPresenter.ts

  frameworks/            ← Ring 4: frameworks & drivers
    react/
      pages/
        OrdersPage.tsx
        CheckoutPage.tsx
      components/
        OrderCard.tsx
        OrderList.tsx
      hooks/
        useOrders.ts
    http/
      apiClient.ts
    storage/
      localStorage.ts

  infrastructure/
    container.ts         ← DI wiring
    config.ts
```

### Making It Work with React Hooks

```typescript
// frameworks/react/hooks/useOrders.ts
// Thin wrapper: connects TanStack Query → use case → presenter

export function useOrders() {
  const { getOrdersUseCase, orderPresenter } = useContainer(); // DI hook

  return useQuery({
    queryKey: ["orders"],
    queryFn: async () => {
      const orders = await getOrdersUseCase.execute({ userId: currentUserId });
      return orderPresenter.presentList(orders); // transform for UI
    },
  });
}

export function usePlaceOrder() {
  const { placeOrderUseCase } = useContainer();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: PlaceOrderInput) => placeOrderUseCase.execute(input),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["orders"] });
      queryClient.invalidateQueries({ queryKey: ["cart"] });
    },
  });
}
```

---

## 9. The Humble Object Pattern

The "Humble Object" makes things that are hard to test humble (simple, thin) and extracts the logic that's worth testing into a separate, easily testable object.

```typescript
// PROBLEM: React component is hard to test because it's tied to React
// Much of the logic in the component is "display logic" — worth testing

// ❌ Hard to test: logic embedded in React component
function OrderCard({ order }: { order: Order }) {
  // Display logic mixed with rendering — hard to unit test
  const isPastDue    = new Date() > new Date(order.dueDate);
  const discountPct  = order.total > 100 ? 10 : 0;
  const displayTotal = order.total * (1 - discountPct / 100);
  const urgentClass  = isPastDue ? 'urgent' : '';

  return <div className={urgentClass}>...</div>;
}

// ✅ Humble Object pattern:
// Extract logic into a pure presenter class (easy to test)
// Leave the component as "humble" (just renders what it's given)

// Presenter: all the display logic (easy to unit test)
class OrderCardPresenter {
  present(order: Order, now = new Date()): OrderCardViewModel {
    const isPastDue   = now > order.dueDate;
    const discountPct = order.total > 100 ? 10 : 0;

    return {
      isPastDue,
      discountPercent: discountPct,
      displayTotal:    order.total * (1 - discountPct / 100),
      urgencyClass:    isPastDue ? 'urgent' : '',
    };
  }
}

// Component: humble — just renders what it receives
function OrderCard({ order }: { order: Order }) {
  const presenter = useMemo(() => new OrderCardPresenter(), []);
  const vm        = presenter.present(order);

  return (
    <div className={vm.urgencyClass}>
      <p>${vm.displayTotal.toFixed(2)}</p>
      {vm.discountPercent > 0 && <span>{vm.discountPercent}% off</span>}
    </div>
  );
}

// Presenter tests: pure unit tests, no React
describe('OrderCardPresenter', () => {
  const presenter = new OrderCardPresenter();

  test('marks order as urgent when past due', () => {
    const pastOrder = { ...mockOrder, dueDate: new Date('2020-01-01') };
    const vm = presenter.present(pastOrder, new Date('2024-01-01'));
    expect(vm.isPastDue).toBe(true);
    expect(vm.urgencyClass).toBe('urgent');
  });
});
```

---

## 10. Screaming Architecture

Uncle Bob's principle: a good architecture "screams" its purpose. When you look at the top-level structure, you should immediately know what the system is about — not what framework it uses.

```
❌ Architecture that screams "React App":
src/
  components/
  hooks/
  contexts/
  redux/
  styles/
  utils/
  → What does this app DO? I can't tell from the structure.

❌ Architecture that screams "MVC Pattern":
src/
  models/
  views/
  controllers/
  → Technical structure, not business structure.

✅ Architecture that screams its business domain:
src/
  orders/
    entities/
    use-cases/
    adapters/
  customers/
  payments/
  inventory/
  shipping/
  → This is clearly an e-commerce system. Framework is invisible at this level.
```

### Business Domain as the Top-Level Organization

```
Clean Architecture folder structure for an e-commerce app:

src/
  ├── orders/           ← "orders" is the loudest thing here
  │   ├── Order.ts      ← entity
  │   ├── PlaceOrderUseCase.ts
  │   ├── CancelOrderUseCase.ts
  │   ├── HttpOrderRepository.ts
  │   └── OrderPresenter.ts
  │
  ├── payments/         ← "payments" — business domain
  │   ├── Payment.ts
  │   ├── ProcessPaymentUseCase.ts
  │   ├── StripeGateway.ts
  │   └── PaymentPresenter.ts
  │
  ├── inventory/        ← "inventory" — business domain
  │   ├── Product.ts
  │   ├── CheckStockUseCase.ts
  │   └── HttpInventoryRepository.ts
  │
  └── frameworks/       ← framework is visible but not dominant
      └── react/
```

---

## 11. Good Practices

### ✅ Keep entities free of external dependencies

```typescript
// ✅ Entity has zero imports from external libraries
// Order.ts
export class Order {
  // Only domain types, no React, no axios, no date-fns
  constructor(private readonly props: OrderProps) {
    this.validateInvariants();
  }
  // ...
}
```

### ✅ Test use cases with test doubles, not mocks of concrete implementations

```typescript
// ✅ Inject interfaces — swap for test doubles in tests
const useCase = new PlaceOrderUseCase(
  new InMemoryOrderRepository(), // implements IOrderRepository
  new FakePaymentGateway(), // implements IPaymentGateway
  new NoOpNotificationService(), // implements INotificationService
);

// Test the behavior, not the framework
const result = await useCase.execute(validInput);
expect(result.order.status).toBe("processing");
```

### ✅ Adapters are the only things that know external formats

```typescript
// ✅ API response mapping isolated in adapter
// No other code sees ApiOrderResponse — only Order entity
class HttpOrderRepository implements IOrderRepository {
  private toApiDto(order: Order): ApiCreateOrderDto { ... }
  private toDomain(api: ApiOrderResponse): Order { ... }
}
```

---

## 12. Bad Practices

### ❌ Fat entities with framework dependencies

```typescript
// ❌ Entity that knows about React state — violates independence
class Order {
  #setState: React.Dispatch<React.SetStateAction<OrderState>>;

  updateStatus(status: OrderStatus) {
    this.#setState((prev) => ({ ...prev, status })); // ← React in entity!
  }
}
```

### ❌ Use cases that call API directly

```typescript
// ❌ Use case bypassing the port/adapter pattern
class PlaceOrderUseCase {
  async execute(input: PlaceOrderInput) {
    // Direct HTTP call — should use injected IOrderRepository
    const response = await fetch("/api/orders", {
      method: "POST",
      body: JSON.stringify(input),
    });
    return response.json();
  }
}
```

### ❌ Presenter logic in entities

```typescript
// ❌ Entity knows about display formatting
class Order {
  getFormattedTotal(): string {
    return `$${this.total.toFixed(2)}`; // display concern in entity!
  }

  getStatusBadgeColor(): string {
    // UI concern in entity!
    return this.status === "completed" ? "green" : "gray";
  }
}
```

---

## 13. Common Mistakes

### Mistake 1 — Over-engineering for small apps

```
Clean Architecture shines at large scale — multiple teams, complex domains.
For a 5-page CRUD app with one developer:
  → The abstraction cost (entities, use cases, ports, adapters, DI container)
     exceeds the benefit of the separation.

Start simpler. Add layers as complexity justifies them.
The pattern to watch for: "I keep changing the API call whenever a business rule changes"
That's the signal to separate domain from infrastructure.
```

### Mistake 2 — Anemic entities (data bags with no behavior)

```typescript
// ❌ Anemic entity: just a data structure, no behavior
interface Order {
  id:     string;
  status: string;
  total:  number;
  items:  OrderItem[];
}

// All logic ends up scattered in services/components:
function canCancelOrder(order: Order): boolean { ... } // in service
function getOrderTotal(order: Order): number { ... }   // in component
function isEligibleForDiscount(order: Order): boolean { ... } // in hook

// ✅ Rich entity: encapsulates its own behavior
class Order {
  canBeCancelled(): boolean { ... }    // belongs here
  get total(): number { ... }           // belongs here
  isEligibleForDiscount(): boolean { ... } // belongs here
}
```

### Mistake 3 — Testing at the wrong level

```typescript
// ❌ E2E test for business rule that should be a unit test
// "orders over $500 get free shipping" tested via Playwright browser test
// → slow, fragile, expensive

// ✅ Business rule tested as unit test on the entity/use case
test("orders over $500 qualify for free shipping", () => {
  const order = new Order({ items: [{ price: 600, quantity: 1 }] });
  expect(order.shippingCost).toBe(0);
});
// → instant, deterministic, no browser, no network
```

---

## 14. Interview-Level Explanation

> **"What is Clean Architecture? How does it differ from regular layered architecture?"**

**Strong answer:**

> "Clean Architecture is a refinement of layered architecture by Robert Martin. Both separate code into horizontal layers with downward dependencies. Clean Architecture adds the Dependency Inversion Principle and the Stable Abstractions Principle to make the separation stronger and more rigorous.
>
> The key difference: in basic layered architecture, the domain layer might still call infrastructure directly — the domain 'just doesn't import React.' In Clean Architecture, the domain doesn't even know the infrastructure exists. Use cases define interfaces (ports) for what they need — 'I need something that saves an Order' — and adapters implement those interfaces. The use case imports only the interface it defined. The implementation can be an HTTP repository, an in-memory test double, a localStorage adapter — swapped without touching the use case.
>
> The four concentric rings: entities (core domain objects with business invariants), use cases (orchestrate operations to fulfill specific application requirements), interface adapters (transform data between use cases and external systems), frameworks and drivers (React, HTTP client, localStorage — the most volatile ring, kept thin). The dependency rule: source code dependencies point inward only. The outermost ring knows about everything inside; the innermost ring knows about nothing outside.
>
> The practical payoff: entities and use cases are completely framework-agnostic. Testing a use case means calling a method with a fake repository and fake payment gateway — no React, no HTTP, no setup. The test runs in milliseconds and tests the real business logic. When Stripe changes its API format, only the StripePaymentGateway adapter changes; the PlaceOrderUseCase and the Order entity don't know anything changed.
>
> The appropriate caveat: this is most valuable for complex domains with long lifetimes. For a simple CRUD app with one developer, the abstraction overhead — entities, use cases, ports, adapters, DI container — exceeds the benefit. The pattern becomes worth it when you have a complex domain, multiple teams, or a system you expect to live for years and change in ways you can't predict."

---

## 15. Exercises

### Exercise 1 — Redesign a use case

The following code is a React hook that mixes concerns. Redesign it following Clean Architecture principles — create an entity, a use case with its port interface, an adapter, and a thin React hook.

```typescript
function useSubscribe(planId: string) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");

  const subscribe = async () => {
    setLoading(true);
    try {
      const user = JSON.parse(localStorage.getItem("user") || "{}");
      if (!user.id) throw new Error("Must be logged in");

      const currentPlan = await fetch(`/api/subscriptions/${user.id}`).then(
        (r) => r.json(),
      );
      if (currentPlan.planId === planId)
        throw new Error("Already on this plan");
      if (currentPlan.status === "past_due")
        throw new Error("Account has unpaid balance");

      await fetch("/api/subscriptions", {
        method: "POST",
        body: JSON.stringify({ userId: user.id, planId }),
      });

      analytics.track("subscription_started", { planId });
      setLoading(false);
    } catch (err) {
      setError(err.message);
      setLoading(false);
    }
  };

  return { subscribe, loading, error };
}
```

<details>
<summary>Solution</summary>

```typescript
// 1. ENTITY: Subscription
class Subscription {
  readonly userId: string;
  readonly planId: string;
  readonly status: "active" | "past_due" | "cancelled";

  constructor(props: SubscriptionProps) {
    this.userId = props.userId;
    this.planId = props.planId;
    this.status = props.status;
  }

  canChangePlan(newPlanId: string): { allowed: boolean; reason?: string } {
    if (this.planId === newPlanId)
      return { allowed: false, reason: "Already on this plan" };
    if (this.status === "past_due")
      return { allowed: false, reason: "Account has unpaid balance" };
    return { allowed: true };
  }
}

// 2. USE CASE PORT
export interface ISubscriptionRepository {
  findByUserId(userId: string): Promise<Subscription | null>;
  save(subscription: Subscription): Promise<Subscription>;
}

// 3. USE CASE
export class StartSubscriptionUseCase {
  constructor(
    private readonly subRepo: ISubscriptionRepository,
    private readonly analytics: IAnalyticsService,
  ) {}

  async execute(userId: string, planId: string): Promise<Subscription> {
    if (!userId) throw new UseCaseError("Must be logged in");

    const current = await this.subRepo.findByUserId(userId);
    if (current) {
      const { allowed, reason } = current.canChangePlan(planId);
      if (!allowed) throw new UseCaseError(reason!);
    }

    const subscription = new Subscription({ userId, planId, status: "active" });
    const saved = await this.subRepo.save(subscription);

    this.analytics.track("subscription_started", { planId });
    return saved;
  }
}

// 4. ADAPTER
export class HttpSubscriptionRepository implements ISubscriptionRepository {
  async findByUserId(userId: string): Promise<Subscription | null> {
    const data = await apiClient.get(`/api/subscriptions/${userId}`);
    return data ? new Subscription(data) : null;
  }

  async save(sub: Subscription): Promise<Subscription> {
    const data = await apiClient.post("/api/subscriptions", {
      userId: sub.userId,
      planId: sub.planId,
    });
    return new Subscription(data);
  }
}

// 5. THIN REACT HOOK (framework layer)
export function useSubscribe(planId: string) {
  const { currentUser } = useAuth();
  const useCase = useContainer().startSubscriptionUseCase;

  return useMutation({
    mutationFn: () => useCase.execute(currentUser.id, planId),
  });
}
```

</details>

---

## 🔗 Related Topics

- [`architecture/01-layered-architecture.md`](./01-layered-architecture.md) — Layered architecture (foundation for Clean Architecture)
- [`architecture/03-domain-driven-design.md`](./03-domain-driven-design.md) — DDD concepts that complement Clean Architecture
- [`system-design/01-large-scale-architecture.md`](../system-design/01-large-scale-architecture.md) — Applying architecture at scale
- [`testing/01-unit-testing.md`](../testing/01-unit-testing.md) — Testing use cases with test doubles
- [`patterns/04-repository.md`](../patterns/04-repository.md) — Repository pattern (used extensively in Clean Architecture)

---

<div align="center">

**Next:** [`architecture/03-domain-driven-design.md`](./03-domain-driven-design.md) →

</div>
