# 03 — Domain-Driven Design

> **"The most important thing in software development is understanding the domain. Not the tools, not the patterns, not the framework. The domain. Everything else follows from that."**
> — Eric Evans

Domain-Driven Design (DDD) is a software development approach that aligns code structure with business reality. Eric Evans introduced it in his 2003 book "Domain-Driven Design: Tackling Complexity in the Heart of Software." While DDD originated with backend/data systems in mind, its vocabulary and tactical patterns are directly applicable to frontend architecture — particularly the concepts of ubiquitous language, bounded contexts, aggregates, and domain events. This document covers the DDD concepts most relevant to frontend engineers and how to apply them in practice.

---

## 📚 Table of Contents

1. [The Core Problem DDD Solves](#1-the-core-problem-ddd-solves)
2. [Ubiquitous Language](#2-ubiquitous-language)
3. [Bounded Contexts](#3-bounded-contexts)
4. [Strategic Design — Context Map](#4-strategic-design--context-map)
5. [Aggregates](#5-aggregates)
6. [Entities vs Value Objects](#6-entities-vs-value-objects)
7. [Domain Events](#7-domain-events)
8. [Repositories in DDD](#8-repositories-in-ddd)
9. [Domain Services](#9-domain-services)
10. [Anti-Corruption Layer](#10-anti-corruption-layer)
11. [DDD on the Frontend — Practical Application](#11-ddd-on-the-frontend--practical-application)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Core Problem DDD Solves

Without deliberate alignment between code and business concepts, large codebases develop a "big ball of mud" — where business logic is scattered across components, utilities, and services with no coherent model, concepts have different names in different parts of the code, and changes to business rules require hunting through 15 files to find where the logic lives.

```
SYMPTOMS OF POOR DOMAIN MODELING:

1. NAMING CONFUSION:
   The product manager calls it "subscription"
   The UI calls it "plan"
   The API returns "membership"
   The database table is "user_products"
   The reducer handles "accountPackage"
   → Nobody knows if these are the same thing

2. SCATTERED LOGIC:
   "Can a user cancel their order?" — this logic is in:
   - OrderCard.tsx (renders cancel button conditionally)
   - useOrders.ts (filters which orders show cancel)
   - orderUtils.ts (canCancelOrder function)
   - OrderService.ts (cancelOrder API call guards)
   → Change the business rule: find and update 4 places

3. ANEMIC MODEL:
   Order is just a TypeScript interface with data
   All logic is in services/utils that take Order as parameters
   No single place that owns "what an Order can do"
```

---

## 2. Ubiquitous Language

Ubiquitous Language is the shared vocabulary between developers and domain experts — the same words used in conversations, documentation, requirements, and code.

### Building the Language

```typescript
// Domain experts talk about:
// "A customer places an order for line items
//  with a shipping address and payment method.
//  Orders can be fulfilled or cancelled.
//  A fulfillment creates a shipment."

// ✅ Code reflects the ubiquitous language:
class Customer { ... }
class Order {
  place(): PlacedOrder { ... }
  fulfil(): Fulfilment { ... }
  cancel(reason: CancellationReason): CancelledOrder { ... }
}
class LineItem { ... }
class ShippingAddress { ... }
class PaymentMethod { ... }
class Fulfilment { ... }
class Shipment { ... }

// ❌ Code that diverges from business language:
class UserProductPurchaseEvent { ... }    // → should be "Order"
class CartLine { ... }                    // → should be "LineItem"
interface CompleteCheckout { ... }        // → should be "PlaceOrder"
function processPaymentTransactionItem()  // → should be "charge"
```

### Ubiquitous Language in TypeScript

```typescript
// Create a glossary as types — the code IS the documentation
// src/domain/shopping/types.ts

/** A customer who has an account and can place orders */
type Customer = { id: CustomerId; email: Email; tier: CustomerTier };

/** A request to purchase items, not yet confirmed */
type Cart = { items: CartItem[]; promoCode?: PromoCode };

/** A confirmed purchase commitment */
type Order = {
  id: OrderId;
  customerId: CustomerId;
  lineItems: LineItem[];
  status: OrderStatus;
};

/** A single product in an order, with quantity and agreed price */
type LineItem = {
  productId: ProductId;
  sku: SKU;
  agreedPrice: Money;
  quantity: Quantity;
};

/** An administrative cancellation of an entire order */
type Cancellation = {
  orderId: OrderId;
  reason: CancellationReason;
  cancelledAt: Date;
};

/** The physical delivery of an order */
type Fulfilment = { orderId: OrderId; shipments: Shipment[] };

/** A package sent to the customer */
type Shipment = {
  trackingNumber: TrackingNumber;
  carrier: Carrier;
  estimatedDelivery: Date;
};
```

---

## 3. Bounded Contexts

A Bounded Context is an explicit boundary within which a domain model is defined and applicable. The same concept can exist in multiple bounded contexts with different meanings.

```
E-COMMERCE SYSTEM — Multiple Bounded Contexts:

┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   CATALOG       │   │   ORDERS        │   │   FULFILLMENT   │
│                 │   │                 │   │                 │
│ Product:        │   │ Product:        │   │ Product:        │
│  - name         │   │  - orderedPrice │   │  - weight       │
│  - description  │   │  - sku          │   │  - dimensions   │
│  - price        │   │  - quantity     │   │  - pickLocation │
│  - images       │   │                 │   │                 │
│  - stock level  │   │ (no images,     │   │ (no price,      │
│  - categories   │   │  no categories) │   │  no description)│
└─────────────────┘   └─────────────────┘   └─────────────────┘

"Product" means different things in each context.
Each context has exactly the Product it needs — no more, no less.
```

### Bounded Contexts on the Frontend

```typescript
// Frontend bounded contexts map to feature modules

// CATALOG CONTEXT: src/features/catalog/
interface CatalogProduct {
  id: ProductId;
  name: string;
  description: string;
  price: Money;
  images: ProductImage[];
  categories: Category[];
  stockStatus: "in-stock" | "low-stock" | "out-of-stock";
}

// ORDER CONTEXT: src/features/orders/
interface OrderProduct {
  productId: ProductId;
  sku: string;
  name: string; // display only — doesn't update if catalog name changes
  orderedPrice: Money; // frozen at time of order
  quantity: number;
}

// FULFILLMENT CONTEXT: src/features/fulfillment/ (admin only)
interface FulfillmentProduct {
  productId: ProductId;
  sku: string;
  weight: Weight;
  dimensions: Dimensions;
  pickLocation: WarehouseLocation;
}

// The same "product" has different shapes in different contexts.
// This is correct — each context gets what it needs.
```

---

## 4. Strategic Design — Context Map

A Context Map shows how bounded contexts relate to each other.

```
RELATIONSHIP TYPES:

PARTNERSHIP:
  Two contexts develop together and support each other.
  Team A and Team B collaborate on the interface between catalog and orders.

CUSTOMER-SUPPLIER:
  One context (supplier) defines an interface; the other (customer) uses it.
  Orders context is the customer; Catalog API is the supplier.
  Customer must adapt when supplier changes.

CONFORMIST:
  The downstream context fully conforms to the upstream model.
  Our frontend conforms to a third-party payment API.
  We have no influence; we adapt to their model.

ANTI-CORRUPTION LAYER (ACL):
  The downstream context translates the upstream model into its own.
  We wrap a legacy API with our own domain model.
  Changes in the legacy API don't leak into our domain.

SHARED KERNEL:
  Two contexts share a subset of the domain model.
  Both the catalog and orders context share the Money value object.
  Changes to the shared kernel require coordination.
```

### Context Map for a Frontend

```
External payment API (Stripe)
        ↓ Conformist: we adapt to Stripe's model
        ↓
   [ACL: StripeAdapter] — translates Stripe concepts to our Payment domain
        ↓
   Payment Bounded Context
        ↓
        ↓ Customer: Payment needs to know about Orders
        ↓
   Orders Bounded Context ←──── Shared Kernel: Money, CustomerId
        ↑
        ↑ Supplier: Catalog provides product info to Orders
        ↑
   Catalog Bounded Context
```

---

## 5. Aggregates

An Aggregate is a cluster of domain objects that must change together, treated as a single unit. Every aggregate has a root entity. All interactions with the aggregate go through the root.

```
AGGREGATE ROOT: Order
  Rules:
    - Only Order can be used to reach LineItems
    - Only Order can add/remove LineItems
    - Changing a LineItem quantity must go through Order
    - Persisting/loading is done on the Order (not individual LineItems)

  WHY:
    Without an aggregate root, two processes could independently
    modify LineItems and violate the invariant:
    "Order total must match sum of LineItem prices × quantities"

    With aggregate root: Order enforces this invariant on every change
```

```typescript
// Order Aggregate
class Order {
  readonly id: OrderId;
  readonly customerId: CustomerId;
  #lineItems: LineItem[];
  #status: OrderStatus;
  #events: DomainEvent[] = [];

  constructor(props: OrderProps) {
    this.id = props.id ?? OrderId.generate();
    this.customerId = props.customerId;
    this.#lineItems = [...props.lineItems];
    this.#status = props.status ?? "draft";
    this.invariants(); // validate on construction
  }

  // Only Order can modify its line items
  addItem(product: CatalogProduct, quantity: Quantity): void {
    this.assertStatus("draft"); // can only add items to draft orders

    const existing = this.#lineItems.find((i) =>
      i.productId.equals(product.id),
    );
    if (existing) {
      const updated = existing.withQuantity(existing.quantity.add(quantity));
      this.#lineItems = this.#lineItems.map((i) =>
        i.productId.equals(product.id) ? updated : i,
      );
    } else {
      this.#lineItems.push(
        LineItem.create({
          productId: product.id,
          sku: product.sku,
          name: product.name,
          agreedPrice: product.currentPrice, // price frozen at order time
          quantity,
        }),
      );
    }

    this.invariants();
  }

  removeItem(productId: ProductId): void {
    this.assertStatus("draft");
    this.#lineItems = this.#lineItems.filter(
      (i) => !i.productId.equals(productId),
    );
    this.invariants();
  }

  place(): void {
    this.assertStatus("draft");
    if (this.#lineItems.length === 0) {
      throw new DomainError("Cannot place an empty order");
    }

    this.#status = "placed";
    this.#events.push(new OrderPlacedEvent(this.id, this.total));
  }

  cancel(reason: CancellationReason): void {
    if (this.#status === "cancelled") {
      throw new DomainError("Order is already cancelled");
    }
    if (this.#status === "delivered") {
      throw new DomainError("Delivered orders cannot be cancelled");
    }

    this.#status = "cancelled";
    this.#events.push(new OrderCancelledEvent(this.id, reason));
  }

  // Aggregate invariants — must always hold
  private invariants(): void {
    if (this.#lineItems.some((i) => i.quantity.value <= 0)) {
      throw new DomainError("All line item quantities must be positive");
    }
    if (this.#lineItems.some((i) => i.agreedPrice.isNegative())) {
      throw new DomainError("Line item prices cannot be negative");
    }
  }

  private assertStatus(expected: OrderStatus): void {
    if (this.#status !== expected) {
      throw new DomainError(
        `Cannot perform operation on ${this.#status} order`,
      );
    }
  }

  // Computed values
  get total(): Money {
    return this.#lineItems.reduce(
      (sum, item) => sum.add(item.agreedPrice.multiply(item.quantity.value)),
      Money.zero("USD"),
    );
  }

  get lineItems(): readonly LineItem[] {
    return Object.freeze([...this.#lineItems]);
  }
  get status(): OrderStatus {
    return this.#status;
  }

  // Collect and clear domain events
  pullEvents(): DomainEvent[] {
    const events = [...this.#events];
    this.#events = [];
    return events;
  }
}
```

---

## 6. Entities vs Value Objects

### Entities — Identity Matters

```typescript
// Entities are identified by ID, not by their attribute values
// Two orders with the same items are DIFFERENT orders

class Order {
  readonly id: OrderId; // identity

  equals(other: Order): boolean {
    return this.id.equals(other.id); // same ID = same order
  }
}

// Two customer objects with same ID = same customer (even if different data)
const customer1 = new Customer({ id: "C-001", email: "old@email.com" });
const customer2 = new Customer({ id: "C-001", email: "new@email.com" });
customer1.equals(customer2); // true — they're the same customer
```

### Value Objects — Value Matters

```typescript
// Value objects are identified by their values — no ID
// Two Money objects with same amount and currency ARE the same

class Money {
  readonly amount: number;
  readonly currency: string;

  constructor(amount: number, currency: string) {
    // Validate on construction — value objects are immutable
    if (amount < 0) throw new EntityError("Amount cannot be negative");
    this.amount = Math.round(amount * 100) / 100;
    this.currency = currency.toUpperCase();
    Object.freeze(this); // immutable
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  add(other: Money): Money {
    if (!this.currency === other.currency) throw new Error("Currency mismatch");
    return new Money(this.amount + other.amount, this.currency);
  }

  // Value objects return new instances — never mutate
  withTax(taxRate: number): Money {
    return new Money(this.amount * (1 + taxRate), this.currency);
  }

  static zero(currency: string): Money {
    return new Money(0, currency);
  }
}

// Common value objects in e-commerce:
class Email {
  readonly value: string;
  constructor(email: string) {
    if (!email.includes("@")) throw new EntityError("Invalid email format");
    this.value = email.toLowerCase().trim();
    Object.freeze(this);
  }
  equals(other: Email) {
    return this.value === other.value;
  }
  toString() {
    return this.value;
  }
}

class Quantity {
  readonly value: number;
  constructor(n: number) {
    if (!Number.isInteger(n) || n < 1)
      throw new EntityError("Quantity must be a positive integer");
    this.value = n;
    Object.freeze(this);
  }
  add(other: Quantity): Quantity {
    return new Quantity(this.value + other.value);
  }
  equals(other: Quantity) {
    return this.value === other.value;
  }
}
```

---

## 7. Domain Events

Domain Events represent something meaningful that happened in the domain. They're facts — past tense, immutable.

```typescript
// Base domain event
abstract class DomainEvent {
  readonly occurredAt: Date;
  readonly eventId: string;

  constructor() {
    this.occurredAt = new Date();
    this.eventId = generateId();
  }
}

// Specific domain events
class OrderPlacedEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly orderTotal: Money,
    readonly customerId: CustomerId,
  ) {
    super();
  }
}

class OrderCancelledEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly reason: CancellationReason,
  ) {
    super();
  }
}

class LineItemAddedEvent extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly productId: ProductId,
    readonly quantity: Quantity,
  ) {
    super();
  }
}
```

### Collecting Events in Aggregates

```typescript
// Aggregate collects events during operations
// Events are published AFTER the aggregate is persisted

class Order {
  #events: DomainEvent[] = [];

  place(): void {
    // ... business logic ...
    this.#status = "placed";
    this.#events.push(
      new OrderPlacedEvent(this.id, this.total, this.customerId),
    );
  }

  // The use case (or repository) collects and dispatches events after saving
  pullEvents(): DomainEvent[] {
    const events = [...this.#events];
    this.#events = [];
    return events;
  }
}

// In the use case:
async function placeOrder(orderId: OrderId): Promise<void> {
  const order = await orderRepo.findById(orderId);
  order.place(); // mutates aggregate, records event internally

  await orderRepo.save(order); // persist FIRST

  // Then publish events (after persistence — guaranteed consistency)
  const events = order.pullEvents();
  for (const event of events) {
    await eventBus.publish(event);
  }
}
```

### Frontend Domain Events vs Application Events

```typescript
// DOMAIN EVENT: something happened in the business domain
class OrderPlacedEvent {
  orderId: OrderId;
  total: Money;
}

// APPLICATION EVENT: something happened in the application
type CartUpdatedEvent = { count: number; total: number }; // UI-level

// Don't conflate them. Domain events are rich with domain meaning.
// Application events are for UI coordination.
```

---

## 8. Repositories in DDD

A Repository provides an illusion of an in-memory collection of domain objects. You ask for objects by criteria; you save objects; you don't write SQL or HTTP calls in your domain.

```typescript
// DDD Repository interface (domain layer — inner ring)
interface OrderRepository {
  // Find
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: CustomerId): Promise<Order[]>;
  findActive(): Promise<Order[]>;

  // Persist
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;

  // Query (specification pattern for complex queries)
  findBySpec(spec: OrderSpecification): Promise<Order[]>;
}

// Specification pattern for complex queries
interface OrderSpecification {
  isSatisfiedBy(order: Order): boolean;
}

class RecentOrdersSpec implements OrderSpecification {
  constructor(private readonly withinDays: number) {}
  isSatisfiedBy(order: Order): boolean {
    const cutoff = new Date();
    cutoff.setDate(cutoff.getDate() - this.withinDays);
    return order.createdAt >= cutoff;
  }
}

class HighValueOrdersSpec implements OrderSpecification {
  constructor(private readonly minTotal: Money) {}
  isSatisfiedBy(order: Order): boolean {
    return order.total.amount >= this.minTotal.amount;
  }
}

// Combine specifications
class AndSpec implements OrderSpecification {
  constructor(private readonly specs: OrderSpecification[]) {}
  isSatisfiedBy(order: Order): boolean {
    return this.specs.every((s) => s.isSatisfiedBy(order));
  }
}

// Usage:
const recentHighValue = new AndSpec([
  new RecentOrdersSpec(30),
  new HighValueOrdersSpec(new Money(500, "USD")),
]);
const orders = await orderRepo.findBySpec(recentHighValue);
```

---

## 9. Domain Services

When a business operation doesn't naturally belong to one entity, it becomes a Domain Service — a stateless service that encapsulates domain logic involving multiple aggregates.

```typescript
// Domain Service: operation spanning multiple aggregates

// "Transfer items from one order to another" — belongs to neither order
class OrderTransferService {
  transfer(
    source: Order,
    destination: Order,
    productId: ProductId,
    quantity: Quantity,
  ): { updatedSource: Order; updatedDestination: Order } {
    // Validate
    const item = source.lineItems.find((i) => i.productId.equals(productId));
    if (!item) throw new DomainError("Product not found in source order");
    if (item.quantity.value < quantity.value) {
      throw new DomainError("Insufficient quantity to transfer");
    }

    // Perform the transfer
    const updatedSource = source.removeItem(productId);
    const updatedDestination = destination.addItem(item.product, quantity);

    return { updatedSource, updatedDestination };
  }
}

// Domain Service: pricing — spans catalog and customer domains
class PricingService {
  calculatePrice(product: CatalogProduct, customer: Customer): Money {
    const basePrice = product.currentPrice;

    // Discount tiers — business rule involving both product and customer
    const discount =
      customer.tier === "gold"
        ? 0.15
        : customer.tier === "silver"
          ? 0.1
          : product.onSale
            ? 0.05
            : 0;

    return basePrice.multiply(1 - discount);
  }
}
```

### Domain Service vs Application Service

```
DOMAIN SERVICE:
  - Pure domain logic
  - Stateless
  - No I/O (no HTTP, no storage)
  - Uses domain objects only
  - Lives in the domain layer

APPLICATION SERVICE (Use Case):
  - Orchestrates domain operations
  - Can have I/O (saves to repo, sends events)
  - Coordinates with infrastructure
  - Lives in the use case layer
```

---

## 10. Anti-Corruption Layer

The ACL translates between an external model (legacy API, third-party SDK) and your domain model, preventing external concepts from leaking into your domain.

```typescript
// External model (Stripe API — we conform to their format)
interface StripePaymentIntent {
  id: string;
  amount: number; // in cents!
  currency: string; // lowercase
  status: "succeeded" | "processing" | "requires_payment_method" | "canceled";
  client_secret: string;
  payment_method: string;
  created: number; // unix timestamp
}

// Our domain model (our language)
interface Payment {
  id: PaymentId;
  amount: Money; // in dollars, proper value object
  status: PaymentStatus; // 'completed' | 'pending' | 'failed' | 'cancelled'
  reference: string; // the Stripe payment intent ID
  paidAt?: Date;
}

// ACL: translates between them (infrastructure layer)
class StripePaymentAdapter {
  toDomain(stripe: StripePaymentIntent): Payment {
    return {
      id: PaymentId.from(stripe.id),
      amount: new Money(stripe.amount / 100, stripe.currency.toUpperCase()),
      status: this.mapStatus(stripe.status),
      reference: stripe.id,
      paidAt:
        stripe.status === "succeeded"
          ? new Date(stripe.created * 1000)
          : undefined,
    };
  }

  toStripeDto(amount: Money): { amount: number; currency: string } {
    return {
      amount: Math.round(amount.amount * 100), // dollars → cents
      currency: amount.currency.toLowerCase(), // uppercase → lowercase
    };
  }

  private mapStatus(
    stripeStatus: StripePaymentIntent["status"],
  ): PaymentStatus {
    const map: Record<typeof stripeStatus, PaymentStatus> = {
      succeeded: "completed",
      processing: "pending",
      requires_payment_method: "failed",
      canceled: "cancelled",
    };
    return map[stripeStatus];
  }
}
```

---

## 11. DDD on the Frontend — Practical Application

### What to Take from DDD

```
STRONGLY APPLICABLE:
  ✓ Ubiquitous language: code uses the same names as the business
  ✓ Bounded contexts: features map to business domains
  ✓ Value objects: Money, Email, Quantity — immutable, validated
  ✓ Entities with behavior: Order.place(), Order.cancel()
  ✓ Anti-corruption layer: wrap third-party APIs/legacy code
  ✓ Domain events: meaningful business events, not UI events

SOMEWHAT APPLICABLE:
  ~ Aggregates: conceptually useful, strict enforcement optional
  ~ Repositories: interface pattern still valuable
  ~ Specifications: useful for complex filtering

LESS APPLICABLE:
  ✗ Full DDD infrastructure (Domain Event Bus, Event Store)
      — usually overkill for pure frontend
  ✗ CQRS (Command Query Responsibility Segregation)
      — complex, only valuable at very large scale
  ✗ Detailed domain modeling sessions (EventStorming)
      — valuable for complex domains with domain experts
```

### Lightweight DDD for React

```typescript
// You don't need the entire DDD apparatus.
// Start with:

// 1. Ubiquitous language in types
type CustomerId = string & { readonly _brand: 'CustomerId' };
type OrderId    = string & { readonly _brand: 'OrderId' };

// 2. Value objects for important concepts
class Money { ... }
class Email { ... }

// 3. Entities with behavior (not just data)
class Order {
  canBeCancelled(): boolean { ... }
  getTotal(): Money { ... }
}

// 4. Feature modules = bounded contexts
src/features/orders/   → Orders BC
src/features/catalog/  → Catalog BC
src/features/payments/ → Payments BC

// 5. ACL for external APIs
class StripeAdapter { toDomain(...): Payment { ... } }

// That's it — you've applied DDD without the full ceremony.
```

---

## 12. Good Practices

### ✅ Use branded types for domain identifiers

```typescript
// Prevent mixing up IDs with plain strings
type OrderId    = string & { readonly _brand: 'OrderId' };
type CustomerId = string & { readonly _brand: 'CustomerId' };
type ProductId  = string & { readonly _brand: 'ProductId' };

// Creates these types safely
const OrderId = {
  create:   (id: string): OrderId => id as OrderId,
  generate: (): OrderId => generateUUID() as OrderId,
};

// TypeScript prevents:
function getOrder(id: OrderId) { ... }
getOrder('customer-123' as CustomerId); // ❌ TypeScript error
getOrder(OrderId.create('order-123'));  // ✅
```

### ✅ Build the ubiquitous language glossary

```typescript
// src/domain/glossary.ts — living documentation of domain terms
// This file IS the ubiquitous language

/**
 * A request to purchase items from a customer.
 * Transitions: draft → placed → processing → shipped → delivered
 *              draft → cancelled (any time before shipping)
 */
type Order = { ... };

/**
 * A single product line in an order, with the price agreed at order time.
 * Note: price is immutable after order is placed, even if product price changes.
 */
type LineItem = { ... };
```

### ✅ Validate at the domain boundary

```typescript
// ✅ Domain objects validate their own invariants on construction
class Order {
  constructor(props: OrderProps) {
    if (!props.customerId) throw new DomainError("Order must have a customer");
    if (props.lineItems.length === 0)
      throw new DomainError("Order must have line items");
    // ...
  }
}
// Invalid orders can never exist — impossible by construction
```

---

## 13. Bad Practices

### ❌ Ignoring bounded context boundaries

```typescript
// ❌ Catalog's full Product object bleeds into Order context
interface OrderLineItem {
  product: CatalogProduct; // ← full catalog product in order context!
  quantity: number;
}
// Now: changing the catalog Product shape breaks Order
// Now: displaying an order requires loading full catalog data

// ✅ Order context uses its own representation
interface OrderLineItem {
  productId: ProductId;
  sku: string;
  name: string; // snapshot at order time
  agreedPrice: Money; // price at order time — immutable
  quantity: number;
}
```

### ❌ Mixing UI language with domain language

```typescript
// ❌ Domain events with UI terminology
orderDomain.emit("buttonClicked");
orderDomain.emit("modalClosed");
orderDomain.emit("formSubmitted");

// ✅ Domain events express business occurrences
orderDomain.emit("orderPlaced");
orderDomain.emit("orderCancelled");
orderDomain.emit("paymentFailed");
```

---

## 14. Common Mistakes

### Mistake 1 — CRUD-first thinking vs domain-first

```typescript
// ❌ CRUD API leaks into domain thinking
interface OrderService {
  createOrder(data: CreateOrderDto): Promise<Order>;
  updateOrder(id: string, data: UpdateOrderDto): Promise<Order>;
  deleteOrder(id: string): Promise<void>;
}
// This is a database wrapper, not a domain service
// "update" and "delete" have no domain meaning

// ✅ Domain operations
interface OrderService {
  placeOrder(
    cart: Cart,
    address: Address,
    payment: PaymentMethod,
  ): Promise<Order>;
  cancelOrder(orderId: OrderId, reason: CancellationReason): Promise<Order>;
  applyPromoCode(orderId: OrderId, code: PromoCode): Promise<Order>;
  processReturn(orderId: OrderId, items: LineItem[]): Promise<Return>;
}
// Domain operations with business meaning
```

### Mistake 2 — Anemic domain model

```typescript
// ❌ Anemic model: just a data container
interface Order {
  id:     string;
  status: string;
  items:  Item[];
  total:  number;
}

// All logic scattered in random places:
// component: if (order.status !== 'delivered' && order.status !== 'cancelled')
// utils:     function canAddItem(order: Order): boolean
// service:   function calculateTotal(order: Order): number
// hook:      if (new Date() - order.createdAt > 30 * 24 * 60 * 60 * 1000)

// ✅ Rich domain model with behavior
class Order {
  canBeCancelled(): boolean  { ... }
  canAcceptItems(): boolean  { ... }
  get total(): Money         { ... }
  isReturnEligible(): boolean { ... }
}
// Logic lives where it belongs — with the data it governs
```

---

## 15. Interview-Level Explanation

> **"What is Domain-Driven Design? How do you apply it to a frontend codebase?"**

**Strong answer:**

> "Domain-Driven Design is an approach that aligns software structure with business reality. Eric Evans introduced it in 2003. Its core insight is that the primary driver of code structure should be the business domain — not the framework, not the database, not the API format.
>
> The most immediately practical concept is Ubiquitous Language: the exact same terminology used by product managers, domain experts, and users should appear in the code. If the product manager talks about 'order fulfilment' and 'line items', those exact words should be types and classes in the codebase. When code uses the same language as the business, conversations between engineers and non-engineers become more productive and translation errors disappear.
>
> Bounded Contexts map naturally to feature modules in the frontend. The concept of 'Product' means different things in the catalog context (name, images, categories, price) versus the order context (SKU, agreed price, quantity at time of order). Having separate Product types in each feature module is correct DDD — not duplication. Each bounded context owns its representation of shared concepts.
>
> The tactical patterns I find most valuable on the frontend are value objects — immutable, self-validating types like Money, Email, Quantity that carry domain meaning and enforce invariants — and entities with behavior, where Order knows how to `place()` itself, `cancel()` itself, and `canAcceptItems()` rather than having that logic scattered across components and utilities.
>
> The Anti-Corruption Layer is valuable whenever you integrate with external APIs. Stripe's payment model uses cents as integers and lowercase currency codes. Your domain model uses dollars as decimals and proper Money value objects. The ACL translates at the boundary so Stripe's conventions never contaminate your domain code.
>
> What I'd leave out for most frontend apps: the full DDD infrastructure (event stores, CQRS) and formal EventStorming workshops. The ubiquitous language, bounded contexts, value objects, and rich entities give you 80% of the benefit with 20% of the ceremony."

---

## 16. Exercises

### Exercise 1 — Identify bounded contexts

An e-learning platform has these features: course catalog, student enrollment, video playback, progress tracking, certificates, payments, discussion forums.

1. Identify at least 4 bounded contexts
2. For one concept that appears in multiple contexts, describe what it looks like in each

<details>
<summary>Solution</summary>

```
Bounded Contexts:

1. CATALOG (course discovery)
   - What's available, searchable, marketable
   - Course: { id, title, description, price, thumbnail, category, instructor, rating, duration }

2. ENROLLMENT (student access management)
   - Who has access to what
   - Course: { id, title, enrolledAt, expiresAt, accessLevel }
   - No description, no price (already paid), no ratings

3. LEARNING (progress and consumption)
   - The actual learning experience
   - Course: { id, sections: Section[], currentPosition, completionPercent }
   - Lesson: { id, title, videoUrl, duration, completed, notes }

4. ASSESSMENT (testing and certification)
   - Evaluating learning outcomes
   - Course: { id, title, passingScore, quizzes: Quiz[], certificate }
   - No video, no pricing, focused on evaluation

5. PAYMENTS (purchase management)
   - Financial transactions
   - Course: { id, title, basePrice, discountedPrice, tax }
   - Very different shape — about money, not learning

"Course" in 5 different contexts:
  CATALOG:    full marketing info, ratings, instructor bio
  ENROLLMENT: just access info, when it expires
  LEARNING:   current position, completed lessons
  ASSESSMENT: quizzes, grades, certificate eligibility
  PAYMENTS:   price, discount, invoice line item

Each context knows only what it needs.
Sharing a single Course type would force every context to
carry data it doesn't use and creates coupling between all contexts.
```

</details>

---

## 🔗 Related Topics

- [`architecture/01-layered-architecture.md`](./01-layered-architecture.md) — Layered structure that DDD sits within
- [`architecture/02-clean-architecture.md`](./02-clean-architecture.md) — Clean Architecture and DDD complement each other
- [`system-design/02-feature-based-structure.md`](../system-design/02-feature-based-structure.md) — Feature modules as bounded contexts
- [`system-design/06-event-driven-frontend.md`](../system-design/06-event-driven-frontend.md) — Domain events in practice
- [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md) — Observer pattern for domain events

---

<div align="center">

**Next:** [`architecture/04-reactive-architecture.md`](./04-reactive-architecture.md) →

</div>
