# 02 — God Components

> **"A god component is a component that knows too much, does too much, and is responsible for too much. It accretes responsibility the way a drain accretes debris: gradually, a piece at a time, each addition seeming reasonable in isolation, until the drain is blocked and everything backs up. The solution isn't cleaning the drain — it's stopping things from falling in."**

A god component is one that has grown so large and owns so many concerns that it can't be understood, tested, or modified without touching many unrelated parts. It typically mixes data fetching, business logic, UI logic, and presentation all in one place, props that span dozens of items, and renders dozens of nested JSX structures. This document covers how god components form, how to recognize them, and systematic decomposition strategies for breaking them apart safely.

---

## 📚 Table of Contents

1. [How God Components Form](#1-how-god-components-form)
2. [Recognizing a God Component](#2-recognizing-a-god-component)
3. [Decomposition Strategy 1 — Extract Custom Hooks](#3-decomposition-strategy-1--extract-custom-hooks)
4. [Decomposition Strategy 2 — Extract Sub-Components](#4-decomposition-strategy-2--extract-sub-components)
5. [Decomposition Strategy 3 — Split by Responsibility](#5-decomposition-strategy-3--split-by-responsibility)
6. [Decomposition Strategy 4 — Container/Presenter Pattern](#6-decomposition-strategy-4--containerpresenter-pattern)
7. [Refactoring Safely — The Strangler Fig Approach](#7-refactoring-safely--the-strangler-fig-approach)
8. [Good Practices](#8-good-practices)
9. [Bad Practices](#9-bad-practices)
10. [Common Mistakes](#10-common-mistakes)
11. [Interview-Level Explanation](#11-interview-level-explanation)
12. [Exercises](#12-exercises)

---

## 1. How God Components Form

```
THE ACCRETION PATTERN:

Day 1: ProductPage renders a product with a title and description. Clean.

Week 2: Add a review section. Paste it into ProductPage. Still manageable.

Month 1: Add related products, an image gallery, add-to-cart logic, inventory
         status, shipping calculator, size selector, color picker, breadcrumb.
         ProductPage is now 600 lines but it "works."

Month 3: Bug in the review sorting. Developer opens ProductPage.js (600 lines),
         finds the reviews section, fixes it. While there, adds recommended
         products. 750 lines now.

Month 6: New feature: cart upsells in the product page. Developer opens the
         1000-line file, can't find where to add it, adds it at the end.

Year 1: ProductPage.js is 1800 lines. Nobody wants to touch it.
        Test coverage is 0% because "it's too complex to mock all those dependencies."
        The last three bugs were caused by side effects between unrelated sections.
```

---

## 2. Recognizing a God Component

```javascript
// METRICS that indicate a god component:
// - Component function > 150 lines
// - More than 10 props
// - More than 5 useState calls
// - More than 5 useEffect calls
// - Renders more than 3-4 levels of JSX nesting
// - Multiple distinct data-fetching operations for unrelated data
// - Passes data to children that it never uses itself (prop drilling indicator)
// - Mixed concerns: fetching + validation + formatting + rendering in one function

// DIAGNOSTIC QUESTIONS:
// "If I wanted to reuse just the review section elsewhere, could I?"
//   → If no: ReviewSection should be its own component
// "What is this component responsible for?"
//   → If the answer is more than one sentence: it's doing too much
// "When this component breaks, how do I find the bug?"
//   → If the answer is "read the whole file": it's too large
// "Can I test this component's business logic without mounting the entire thing?"
//   → If no: logic should be extracted to hooks/utilities
```

### An Archetypal God Component

```jsx
// ❌ The god component — everything crammed into one function
function ProductPage({ productId }) {
  // State: multiple unrelated concerns
  const [product, setProduct] = useState(null);
  const [reviews, setReviews] = useState([]);
  const [relatedProducts, setRelated] = useState([]);
  const [selectedImage, setSelectedImage] = useState(0);
  const [selectedSize, setSelectedSize] = useState(null);
  const [selectedColor, setSelectedColor] = useState(null);
  const [quantity, setQuantity] = useState(1);
  const [isWishlisted, setWishlisted] = useState(false);
  const [reviewSort, setReviewSort] = useState("newest");
  const [shippingZip, setShippingZip] = useState("");
  const [shippingResult, setShipping] = useState(null);
  const [isCartAdding, setCartAdding] = useState(false);
  const [cartError, setCartError] = useState(null);
  const [reviewPage, setReviewPage] = useState(1);
  const [isDescExpanded, setDescExpanded] = useState(false);

  // Effects: unrelated fetches all in one component
  useEffect(() => {
    fetch(`/api/products/${productId}`)
      .then((r) => r.json())
      .then(setProduct);
  }, [productId]);

  useEffect(() => {
    if (product)
      fetch(
        `/api/products/${productId}/reviews?sort=${reviewSort}&page=${reviewPage}`,
      )
        .then((r) => r.json())
        .then(setReviews);
  }, [productId, product, reviewSort, reviewPage]);

  useEffect(() => {
    if (product)
      fetch(`/api/products/related/${product.categoryId}`)
        .then((r) => r.json())
        .then(setRelated);
  }, [product]);

  useEffect(() => {
    fetch(`/api/wishlist/${productId}/status`)
      .then((r) => r.json())
      .then(({ wishlisted }) => setWishlisted(wishlisted));
  }, [productId]);

  // Event handlers: business logic mixed with UI concerns
  async function handleAddToCart() {
    if (!selectedSize) {
      setCartError("Please select a size");
      return;
    }
    if (!selectedColor) {
      setCartError("Please select a color");
      return;
    }
    setCartAdding(true);
    setCartError(null);
    try {
      await fetch("/api/cart", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          productId,
          size: selectedSize,
          color: selectedColor,
          quantity,
        }),
      });
      // track analytics
      window.gtag("event", "add_to_cart", { productId, value: product.price });
    } catch (e) {
      setCartError("Failed to add to cart");
    } finally {
      setCartAdding(false);
    }
  }

  async function checkShipping() {
    const result = await fetch(
      `/api/shipping?zip=${shippingZip}&productId=${productId}`,
    ).then((r) => r.json());
    setShipping(result);
  }

  function handleWishlist() {
    setWishlisted((w) => !w);
    fetch(`/api/wishlist/${productId}`, {
      method: isWishlisted ? "DELETE" : "POST",
    });
  }

  // Formatting utilities
  function formatPrice(price) {
    return `$${(price / 100).toFixed(2)}`;
  }
  function formatReviewDate(date) {
    return new Date(date).toLocaleDateString();
  }
  function getStockStatus() {
    /* ... */
  }
  function calculateDiscount() {
    /* ... */
  }

  if (!product) return <Spinner />;

  // 100+ lines of JSX mixing all of the above
  return (
    <div className="product-page">{/* ... 100+ lines of mixed JSX ... */}</div>
  );
}
// Result: 300+ lines, untestable, unmaintainable
```

---

## 3. Decomposition Strategy 1 — Extract Custom Hooks

The first pass: extract the data-fetching and business logic into hooks. The component becomes a thin coordinator:

```typescript
// Hook: product data
function useProduct(productId: string) {
  return useQuery({
    queryKey: ['product', productId],
    queryFn:  () => productsApi.get(productId),
  });
}

// Hook: reviews with sort and pagination
function useProductReviews(productId: string) {
  const [sort, setSort]   = useState<'newest' | 'rating'>('newest');
  const [page, setPage]   = useState(1);

  const query = useQuery({
    queryKey: ['reviews', productId, sort, page],
    queryFn:  () => reviewsApi.list({ productId, sort, page }),
  });

  return { ...query, sort, setSort, page, setPage };
}

// Hook: add to cart logic
function useAddToCart(productId: string) {
  const [error, setError] = useState<string | null>(null);

  const mutation = useMutation({
    mutationFn: (cartItem: CartItem) => cartApi.add(cartItem),
    onError:    () => setError('Failed to add to cart'),
    onSuccess:  (_, vars) => {
      analytics.track('add_to_cart', { productId, ...vars });
    },
  });

  function addToCart(size: string, color: string, quantity: number) {
    if (!size)  { setError('Please select a size');  return; }
    if (!color) { setError('Please select a color'); return; }
    setError(null);
    mutation.mutate({ productId, size, color, quantity });
  }

  return { addToCart, isAdding: mutation.isPending, error };
}

// Hook: wishlist
function useWishlist(productId: string) {
  const queryClient = useQueryClient();

  const { data: isWishlisted } = useQuery({
    queryKey: ['wishlist', productId],
    queryFn:  () => wishlistApi.getStatus(productId),
  });

  const { mutate: toggle } = useMutation({
    mutationFn: () => isWishlisted ? wishlistApi.remove(productId) : wishlistApi.add(productId),
    onMutate:   async () => {
      await queryClient.cancelQueries({ queryKey: ['wishlist', productId] });
      queryClient.setQueryData(['wishlist', productId], !isWishlisted);
    },
  });

  return { isWishlisted: !!isWishlisted, toggle };
}

// NOW the component is thin:
function ProductPage({ productId }) {
  const product      = useProduct(productId);
  const reviews      = useProductReviews(productId);
  const addToCart    = useAddToCart(productId);
  const wishlist     = useWishlist(productId);

  if (product.isLoading) return <Spinner />;
  if (product.isError)   return <ErrorMessage />;

  return (
    <div className="product-page">
      <ProductGallery images={product.data.images} />
      <ProductInfo product={product.data} wishlist={wishlist} addToCart={addToCart} />
      <ProductReviews reviews={reviews} />
      <RelatedProducts categoryId={product.data.categoryId} />
    </div>
  );
}
```

---

## 4. Decomposition Strategy 2 — Extract Sub-Components

After extracting hooks, extract visual sub-components, each with a single clear purpose:

```jsx
// Each sub-component is independently understandable and testable

function ProductGallery({ images }) {
  const [activeIndex, setActiveIndex] = useState(0);
  return (
    <div className="gallery">
      <img
        src={images[activeIndex].large}
        alt={`Product view ${activeIndex + 1}`}
      />
      <div className="gallery-thumbs">
        {images.map((img, i) => (
          <img
            key={img.id}
            src={img.thumb}
            onClick={() => setActiveIndex(i)}
            className={i === activeIndex ? "active" : ""}
          />
        ))}
      </div>
    </div>
  );
}

function ProductInfo({ product, wishlist, addToCart }) {
  const [selectedSize, setSelectedSize] = useState(null);
  const [selectedColor, setSelectedColor] = useState(null);
  const [quantity, setQuantity] = useState(1);

  return (
    <div className="product-info">
      <h1>{product.name}</h1>
      <ProductPrice
        price={product.price}
        originalPrice={product.originalPrice}
      />
      <SizeSelector
        sizes={product.sizes}
        value={selectedSize}
        onChange={setSelectedSize}
      />
      <ColorSelector
        colors={product.colors}
        value={selectedColor}
        onChange={setSelectedColor}
      />
      <QuantityInput
        value={quantity}
        onChange={setQuantity}
        max={product.stock}
      />
      {addToCart.error && <ErrorMessage message={addToCart.error} />}
      <Button
        onClick={() =>
          addToCart.addToCart(selectedSize, selectedColor, quantity)
        }
        loading={addToCart.isAdding}
      >
        Add to Cart
      </Button>
      <WishlistButton
        isWishlisted={wishlist.isWishlisted}
        onToggle={wishlist.toggle}
      />
    </div>
  );
}

function ProductReviews({ reviews }) {
  return (
    <section aria-label="Customer reviews">
      <ReviewSortControls sort={reviews.sort} onSortChange={reviews.setSort} />
      <ReviewList
        reviews={reviews.data?.items ?? []}
        isLoading={reviews.isLoading}
      />
      <Pagination
        page={reviews.page}
        totalPages={reviews.data?.totalPages ?? 1}
        onChange={reviews.setPage}
      />
    </section>
  );
}

function RelatedProducts({ categoryId }) {
  // Self-contained: fetches its own data
  const { data: products } = useQuery({
    queryKey: ["related", categoryId],
    queryFn: () => productsApi.getByCategory(categoryId, { limit: 4 }),
  });

  return (
    <section>
      <h2>You may also like</h2>
      <ProductGrid products={products ?? []} />
    </section>
  );
}
```

---

## 5. Decomposition Strategy 3 — Split by Responsibility

Apply the Single Responsibility Principle: each component should have one reason to change.

```
RESPONSIBILITIES TO SEPARATE:

Data fetching / server state:
  → Custom hooks with useQuery/useMutation
  → Self-contained components that fetch their own data

Business logic (validation, calculations, transformations):
  → Pure functions in utility files
  → Custom hooks for stateful business logic

UI state (selections, open/closed, active tab):
  → State in the smallest component that needs it

Presentation (rendering, styling):
  → "Dumb" components that receive props and render

Side effects (analytics, error logging, URL updates):
  → useEffect in dedicated hooks (useAnalytics, useUrlSync)
```

```typescript
// ✅ Business logic extracted to pure functions — testable without mounting
// utils/product.ts
export function calculateDiscount(
  price: number,
  originalPrice: number,
): number {
  return Math.round((1 - price / originalPrice) * 100);
}

export function getStockStatus(
  stock: number,
): "in_stock" | "low_stock" | "out_of_stock" {
  if (stock === 0) return "out_of_stock";
  if (stock <= 5) return "low_stock";
  return "in_stock";
}

export function formatPrice(cents: number, currency = "USD"): string {
  return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(
    cents / 100,
  );
}

// These can be unit tested without React:
test("calculateDiscount", () => {
  expect(calculateDiscount(80, 100)).toBe(20); // 20% off
});
```

---

## 6. Decomposition Strategy 4 — Container/Presenter Pattern

Separate components that know about data sources from components that render:

```jsx
// PRESENTER: pure rendering, no data fetching, easily testable
function ProductCardPresenter({
  title,
  price,
  imageUrl,
  rating,
  reviewCount,
  onAddToCart,
  onWishlist,
  isWishlisted,
  isAddingToCart,
}) {
  return (
    <article className="product-card">
      <img src={imageUrl} alt={title} />
      <h3>{title}</h3>
      <p className="price">{price}</p>
      <Rating value={rating} count={reviewCount} />
      <button onClick={onAddToCart} disabled={isAddingToCart}>
        {isAddingToCart ? "Adding..." : "Add to Cart"}
      </button>
      <WishlistButton active={isWishlisted} onClick={onWishlist} />
    </article>
  );
}

// CONTAINER: knows about data, wires up behavior
function ProductCard({ productId }) {
  const { data: product } = useProduct(productId);
  const { isWishlisted, toggle } = useWishlist(productId);
  const { addToCart, isAdding } = useAddToCart(productId);

  if (!product) return <ProductCardSkeleton />;

  return (
    <ProductCardPresenter
      title={product.name}
      price={formatPrice(product.price)}
      imageUrl={product.images[0].thumb}
      rating={product.rating}
      reviewCount={product.reviewCount}
      isWishlisted={isWishlisted}
      isAddingToCart={isAdding}
      onAddToCart={() =>
        addToCart(product.defaultSize, product.defaultColor, 1)
      }
      onWishlist={toggle}
    />
  );
}

// BENEFIT: ProductCardPresenter can be tested with mock data in Storybook
// and unit tests without any network or React Query setup
```

---

## 7. Refactoring Safely — The Strangler Fig Approach

When you can't rewrite a god component all at once, decompose incrementally:

```
THE STRANGLER FIG STRATEGY:

1. EXTRACT HOOKS FIRST (non-breaking):
   Move all useState/useEffect into custom hooks.
   The component structure doesn't change — it just calls hooks instead of
   having the logic inline. Tests should still pass.

2. EXTRACT LEAF COMPONENTS (non-breaking):
   Find the deepest, most self-contained JSX sections and extract them.
   Start with things that could clearly be reused elsewhere (ImageGallery, Rating).
   The parent component passes props down — same output, less code per file.

3. EXTRACT MID-LEVEL COMPONENTS:
   Extract the next level up. ProductInfo, ProductReviews sections.
   These now receive props that were previously inline state.

4. COLOCATE STATE:
   Move state that was lifted "just in case" back down to the component
   that actually owns it.

5. ADD TESTS AFTER EACH STEP:
   Each extracted hook and component should get unit tests immediately.
   You can't add tests to a 1000-line god component, but you CAN add tests
   to a 30-line hook.

DO NOT:
  - Rewrite everything at once (high risk, high conflict with other work)
  - Extract components without extracting their state (creates prop drilling)
  - Extract for extraction's sake (only extract when there's a clear reason)
```

---

## 8. Good Practices

### ✅ Set a line limit on component files as a forcing function

```
Team rule: no component file > 200 lines.
When a component approaches the limit: that's the signal to decompose.
The limit itself is less important than having a limit — it forces the
conversation before the debt accumulates.
```

### ✅ Name components by what they RENDER, not by the page they appear on

```jsx
// ❌ Page-named components accrete everything related to that page
function CheckoutPage() { /* 800 lines */ }

// ✅ Role-named components have a natural scope
function CheckoutOrderSummary() { /* 80 lines */ }
function CheckoutPaymentForm()   { /* 120 lines */ }
function CheckoutAddressForm()   { /* 100 lines */ }
function CheckoutPage() {
  return (/* assembles the above */);
}
```

### ✅ Extract business logic to pure functions immediately

```typescript
// ✅ Pure functions: testable, reusable, no React needed
export const calculateTax = (subtotal, rate) => subtotal * rate;
export const formatOrderId = (id) => `ORD-${id.toString().padStart(8, "0")}`;
// Add unit tests right away — these are trivial to test
```

---

## 9. Bad Practices

### ❌ Using a single file because "it's easier to find things"

```
Argument: "I put everything in one file so I always know where to look."
Counter: Searching within a 1000-line file is harder than searching across
         well-named smaller files. IDEs can jump to definitions across files.
```

### ❌ Extracting components without their state (accidental prop drilling)

```jsx
// ❌ Extract the JSX but leave the state in the parent → prop drilling
function ProductPage() {
  const [selectedSize, setSelectedSize] = useState(null); // stays in parent
  // ...
  return <SizeSelector selectedSize={selectedSize} onChange={setSelectedSize} />;
}
// SizeSelector doesn't own its state — ProductPage still manages it

// ✅ Extract the component WITH its state
function SizeSelector({ sizes, onSelect }) {
  const [selected, setSelected] = useState(null); // owns its own state
  return (/* ... */);
}
```

---

## 10. Common Mistakes

### Mistake 1 — Premature decomposition (extracting too early)

```jsx
// ❌ Extracting a 10-line section into its own file/component
// before it has any clear reuse case or independent reason to change
function ProductPrice({ price }) {
  return <span>${(price / 100).toFixed(2)}</span>;
}
// This should stay inline in the parent until it grows or is reused

// The right time to extract:
// 1. The logic is reused in multiple places
// 2. The section has grown large enough to be hard to understand in context
// 3. The section changes independently from its neighbors
// 4. It needs independent testing
```

### Mistake 2 — Naming extracted components with generic names

```jsx
// ❌ Generic names that don't constrain scope
function Section({ children }) { ... }
function Container({ children }) { ... }
function Wrapper({ children }) { ... }
// These attract responsibility: "it's just a Section, add it here"

// ✅ Specific names that communicate single purpose
function ProductReviewSection({ productId }) { ... }
function ShoppingCartContainer({ items }) { ... }
function ModalOverlayWrapper({ onClose, children }) { ... }
```

---

## 11. Interview-Level Explanation

> **"What are god components and how do you refactor them?"**

**Strong answer:**

> "A god component is one that has accumulated too many responsibilities — it fetches data, runs business logic, manages UI state, formats data, handles side effects, and renders a large, deeply nested JSX tree, all in one function. They form gradually through accretion: each addition is a small, reasonable decision, but over months the file becomes unmaintainable. Nobody wants to touch it, test coverage drops to zero because it's too complex to mock all the dependencies, and every bug fix risks breaking unrelated features.
>
> I recognize god components by a few signals: the component function is over 150 lines, there are more than five or six `useState` or `useEffect` calls, multiple distinct data-fetching operations for unrelated data, and business logic mixed directly with rendering. The real diagnostic is whether I can describe the component's responsibility in one sentence — if I can't, it's doing too much.
>
> The refactoring has four moves, which I usually apply in sequence. First, extract custom hooks — pull all the data fetching and stateful logic out of the component into focused hooks like `useProductData`, `useReviews`, `useAddToCart`. The component becomes a thin coordinator that calls hooks and passes their outputs to sub-components. This immediately makes the logic independently testable.
>
> Second, extract sub-components — identify the visually distinct sections of the JSX (image gallery, product info, reviews section, related products) and pull each into its own component. Each sub-component gets the state it needs either from the hooks extracted in step one or manages its own internal state.
>
> Third, extract pure functions — business logic like price formatting, discount calculation, and validation belongs in utility files, not in components. Pure functions are trivially testable.
>
> Fourth, colocate state — if state was lifted up to the god component for no particular reason, move it back down into the smallest component that actually needs it.
>
> I use an incremental 'strangler fig' approach when the component is large: extract hooks first (non-breaking, tests still pass), then leaf components, then mid-level components. Add tests at each step. Never try to rewrite the whole thing at once."

---

## 12. Exercises

### Exercise 1 — Decompose this component

```jsx
// This UserSettings component does too much. Identify the separate concerns
// and sketch the decomposition: which hooks and components should be extracted?

function UserSettings({ userId }) {
  const [profile, setProfile]         = useState(null);
  const [name, setName]               = useState('');
  const [email, setEmail]             = useState('');
  const [bio, setBio]                 = useState('');
  const [avatar, setAvatar]           = useState(null);
  const [notifications, setNotifs]    = useState({});
  const [theme, setTheme]             = useState('light');
  const [language, setLanguage]       = useState('en');
  const [isSaving, setSaving]         = useState(false);
  const [saveError, setSaveError]     = useState(null);
  const [activeTab, setTab]           = useState('profile');
  const [confirmDelete, setConfirmDel] = useState(false);

  useEffect(() => {
    fetch(`/api/users/${userId}`).then(r => r.json()).then(user => {
      setProfile(user);
      setName(user.name);
      setEmail(user.email);
      setBio(user.bio);
    });
    fetch(`/api/users/${userId}/notifications`).then(r => r.json()).then(setNotifs);
    fetch(`/api/users/${userId}/preferences`).then(r => r.json()).then(prefs => {
      setTheme(prefs.theme);
      setLanguage(prefs.language);
    });
  }, [userId]);

  async function saveProfile() { /* ... */ }
  async function saveNotifications() { /* ... */ }
  async function savePreferences() { /* ... */ }
  async function deleteAccount() { /* ... */ }

  return (
    <div>
      <Tabs activeTab={activeTab} onChange={setTab} />
      {activeTab === 'profile' && (/* 50 lines of profile JSX */)}
      {activeTab === 'notifications' && (/* 40 lines of notification JSX */)}
      {activeTab === 'preferences' && (/* 30 lines of preference JSX */)}
      {activeTab === 'danger' && (/* 20 lines of danger zone JSX */)}
    </div>
  );
}
```

<details>
<summary>Solution</summary>

```
DECOMPOSITION PLAN:

IDENTIFIED CONCERNS (each should be independent):
1. Profile data + editing (name, email, bio, avatar)
2. Notification preferences
3. App preferences (theme, language)
4. Account deletion
5. Tab navigation state

HOOKS TO EXTRACT:
  useProfile(userId)       → fetches/saves profile data + form state
  useNotifications(userId) → fetches/saves notification prefs + form state
  usePreferences(userId)   → fetches/saves app preferences + form state

COMPONENTS TO EXTRACT:
  ProfileSettingsTab       → renders profile form, uses useProfile()
  NotificationSettingsTab  → renders notification form, uses useNotifications()
  PreferencesTab           → renders preferences form, uses usePreferences()
  DangerZoneTab            → renders delete account with confirmation dialog

RESULT:
  function UserSettings({ userId }) {
    const [activeTab, setTab] = useState('profile');

    return (
      <div>
        <SettingsTabs value={activeTab} onChange={setTab} />
        {activeTab === 'profile'       && <ProfileSettingsTab userId={userId} />}
        {activeTab === 'notifications' && <NotificationSettingsTab userId={userId} />}
        {activeTab === 'preferences'   && <PreferencesTab userId={userId} />}
        {activeTab === 'danger'        && <DangerZoneTab userId={userId} />}
      </div>
    );
  }

  Each tab:
  - Manages its own data fetching
  - Manages its own form state
  - Has its own save logic
  - Can be tested in isolation
  - Can be developed/modified independently

  UserSettings is now only responsible for:
  - Tab navigation state (15 lines)
  - Routing to the correct tab component

  If we want to add a new settings section: add a new tab component,
  add it to the switch — UserSettings doesn't change at all.
```

</details>

---

## 🔗 Related Topics

- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md) — Extracting logic into hooks
- [`patterns/01-component-composition.md`](../patterns/01-component-composition.md) — Composing smaller components
- [`anti-patterns/01-prop-drilling.md`](./01-prop-drilling.md) — Prop drilling often causes god components
- [`system-design/02-feature-based-structure.md`](../system-design/02-feature-based-structure.md) — Feature-based file organization

---

<div align="center">

**Next:** [`anti-patterns/03-premature-optimization.md`](./03-premature-optimization.md) →

</div>
