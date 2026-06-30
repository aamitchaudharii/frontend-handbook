# 02 — Project: E-Commerce Product Page

> **"A product page looks simple from the outside — an image, a price, an add-to-cart button — but it sits at the intersection of nearly every frontend concern: SEO (it must rank and preview well), performance (it's the highest-traffic page type on most e-commerce sites), state complexity (variants, inventory, pricing rules), and conversion-critical UX. Getting it right requires holding all of these simultaneously."**

This project guide builds a complete product detail page: image gallery, variant selection (size/color) with inventory awareness, dynamic pricing, reviews with pagination, related products, and add-to-cart with optimistic feedback — all while meeting strict performance and SEO requirements.

---

## 📚 What You'll Build

A product page with: an interactive image gallery (zoom, thumbnails), variant selectors that update price/availability/images dynamically, a sticky add-to-cart bar, paginated reviews with filtering, related products carousel, and full SEO metadata — built to hit Core Web Vitals targets on mobile.

---

## Requirements

```
FUNCTIONAL:
  - Image gallery: main image + thumbnails, zoom on hover/tap
  - Variant selection: size, color — selecting a variant updates price,
    available sizes (some combinations may be out of stock), and images
  - Add to cart: quantity selector, stock validation, optimistic UI
  - Reviews: paginated, sortable (newest/highest/lowest rated), with photos
  - Related products: horizontally scrollable carousel
  - Breadcrumb navigation

NON-FUNCTIONAL:
  - LCP under 2.5s on mobile (likely the main product image)
  - SEO: server-rendered with structured data (Product schema)
  - No layout shift when variant selection changes price/availability
  - Works correctly when navigating directly to a specific variant via URL
    (e.g., /products/shoe-123?color=red&size=10)
```

---

## Architecture Overview

```
COMPONENT TREE:
  <ProductPage>
    <Breadcrumbs />
    <ProductGallery>            (image carousel + zoom)
      <MainImage />
      <ThumbnailStrip />
    <ProductInfo>
      <ProductTitle />
      <PriceDisplay />          (reflects selected variant)
      <VariantSelector>          (size, color)
        <ColorSwatches />
        <SizeButtons />
      <StockStatus />            (in stock / low stock / out of stock)
      <QuantitySelector />
      <AddToCartButton />
    <ProductTabs>                (description, specs, shipping info)
    <ReviewsSection>
      <ReviewSummary />          (average rating, distribution chart)
      <ReviewFilters />
      <ReviewList />             (paginated)
    <RelatedProducts />

DATA MODEL:
  Product: { id, name, basePrice, images: Image[], variants: Variant[] }
  Variant: { id, color, size, price, stock, imageIndex }

  KEY INSIGHT: a "product" has multiple "variants" — each variant is a
  specific (color, size) combination with its OWN price and stock level.
  The page must derive available options FROM the variant data, not treat
  color/size as independent dropdowns (some combos may not exist).
```

---

## Step 1 — Variant Selection Logic (The Core Complexity)

The hardest part of this page isn't the UI — it's correctly deriving which options are selectable based on the current selection and the variant matrix.

```typescript
interface Variant {
  id: string;
  color: string;
  size: string;
  price: number;
  stock: number;
}

function useVariantSelector(
  variants: Variant[],
  initialSelection?: { color?: string; size?: string },
) {
  const [selectedColor, setSelectedColor] = useState(
    initialSelection?.color ?? variants[0]?.color,
  );
  const [selectedSize, setSelectedSize] = useState(initialSelection?.size);

  // Derive: which colors exist at all (regardless of size)
  const availableColors = useMemo(
    () => [...new Set(variants.map((v) => v.color))],
    [variants],
  );

  // Derive: which sizes are available FOR THE SELECTED COLOR
  const availableSizes = useMemo(
    () => [
      ...new Set(
        variants.filter((v) => v.color === selectedColor).map((v) => v.size),
      ),
    ],
    [variants, selectedColor],
  );

  // If the currently selected size doesn't exist for the new color: clear it
  useEffect(() => {
    if (selectedSize && !availableSizes.includes(selectedSize)) {
      setSelectedSize(undefined);
    }
  }, [selectedColor, availableSizes, selectedSize]);

  // The fully resolved variant (only exists once BOTH color and size are chosen)
  const selectedVariant = useMemo(
    () =>
      variants.find(
        (v) => v.color === selectedColor && v.size === selectedSize,
      ),
    [variants, selectedColor, selectedSize],
  );

  // Helper: is a given size disabled for the CURRENTLY selected color?
  // (used to show "out of stock" styling on size buttons, not hide them)
  function isSizeAvailable(size: string): boolean {
    const variant = variants.find(
      (v) => v.color === selectedColor && v.size === size,
    );
    return !!variant && variant.stock > 0;
  }

  function isColorAvailable(color: string): boolean {
    return variants.some((v) => v.color === color && v.stock > 0);
  }

  return {
    selectedColor,
    setSelectedColor,
    selectedSize,
    setSelectedSize,
    selectedVariant,
    availableColors,
    availableSizes,
    isSizeAvailable,
    isColorAvailable,
  };
}
```

**Key decision:** sizes that are out-of-stock for the current color selection are shown but DISABLED (with strikethrough styling), not hidden. Hiding them is a common mistake — it makes the size grid jump around and confuses users about whether a size exists at all for the product. Showing-but-disabled communicates "this combination exists, but it's currently unavailable."

---

## Step 2 — URL Sync for Variant Selection (Deep Linking + Sharing)

```typescript
function useVariantUrlSync(
  selectedColor: string | undefined,
  selectedSize: string | undefined,
) {
  const [searchParams, setSearchParams] = useSearchParams();

  // Initialize from URL on mount
  const initialSelection = useMemo(
    () => ({
      color: searchParams.get("color") ?? undefined,
      size: searchParams.get("size") ?? undefined,
    }),
    [],
  ); // intentionally empty deps — only read URL once on mount

  // Sync selection changes back to the URL (without adding history entries)
  useEffect(() => {
    const params = new URLSearchParams(searchParams);
    if (selectedColor) params.set("color", selectedColor);
    else params.delete("color");
    if (selectedSize) params.set("size", selectedSize);
    else params.delete("size");
    setSearchParams(params, { replace: true }); // replace, not push — avoid history spam
  }, [selectedColor, selectedSize]);

  return initialSelection;
}
```

**Key decision:** `{ replace: true }` is essential here — without it, every variant click pushes a new browser history entry, making the back button cycle through every color/size combination the user tried instead of navigating away from the page. This is a frequently-missed detail that creates a frustrating back-button experience.

---

## Step 3 — Layout-Shift-Free Price and Stock Updates

```jsx
function PriceDisplay({ variant, basePrice }) {
  const price = variant?.price ?? basePrice;
  const hasDiscount = variant?.originalPrice && variant.originalPrice > price;

  return (
    // Reserve space for the discount badge even when not present,
    // to avoid layout shift when switching between discounted/non-discounted variants
    <div className="price-display" style={{ minHeight: "3rem" }}>
      <span className="price-current">{formatPrice(price)}</span>
      {hasDiscount && (
        <>
          <span className="price-original">
            {formatPrice(variant.originalPrice)}
          </span>
          <span className="price-discount-badge">
            {Math.round((1 - price / variant.originalPrice) * 100)}% off
          </span>
        </>
      )}
    </div>
  );
}

function StockStatus({ variant }) {
  if (!variant) {
    return (
      <p className="stock-status stock-status--neutral">
        Select size and color
      </p>
    );
  }
  if (variant.stock === 0) {
    return <p className="stock-status stock-status--out">Out of stock</p>;
  }
  if (variant.stock <= 5) {
    return (
      <p className="stock-status stock-status--low">
        Only {variant.stock} left!
      </p>
    );
  }
  return <p className="stock-status stock-status--in">In stock</p>;
}
```

**Key decision:** the price container has a reserved `minHeight` so that switching from a discounted variant to a non-discounted one (which removes the badge) doesn't shift the layout below it. This is a small detail with outsized impact on Cumulative Layout Shift (CLS) scores, since variant switching happens frequently during a typical browsing session.

---

## Step 4 — Image Gallery with Variant-Aware Images

```jsx
function ProductGallery({ images, selectedVariant }) {
  const [activeIndex, setActiveIndex] = useState(0);

  // When the variant changes (e.g., user picks a different color), jump
  // to that variant's primary image, if it has one
  useEffect(() => {
    if (selectedVariant?.imageIndex !== undefined) {
      setActiveIndex(selectedVariant.imageIndex);
    }
  }, [selectedVariant?.id]);

  return (
    <div className="gallery">
      <div className="gallery-main">
        {/* LCP candidate: this image must be optimized aggressively */}
        <img
          src={images[activeIndex].large}
          srcSet={`${images[activeIndex].large} 1x, ${images[activeIndex].large2x} 2x`}
          alt={images[activeIndex].alt}
          width={800}
          height={800}
          fetchPriority={activeIndex === 0 ? "high" : "auto"}
          loading={activeIndex === 0 ? "eager" : "lazy"}
        />
      </div>
      <div className="gallery-thumbs">
        {images.map((img, i) => (
          <button
            key={img.id}
            onClick={() => setActiveIndex(i)}
            className={i === activeIndex ? "active" : ""}
            aria-label={`View image ${i + 1}`}
          >
            <img src={img.thumb} alt="" loading="lazy" width={64} height={64} />
          </button>
        ))}
      </div>
    </div>
  );
}
```

**Key decision:** the first/main image gets `fetchPriority="high"` and `loading="eager"` because it's almost certainly the LCP (Largest Contentful Paint) element on this page — this single attribute can meaningfully improve the LCP metric by telling the browser to prioritize this image's network request over other resources. All other images lazy-load.

---

## Step 5 — Reviews with Server-Driven Pagination and Filtering

```typescript
function useReviews(productId: string) {
  const [sort, setSort]     = useState<'newest' | 'highest' | 'lowest'>('newest');
  const [ratingFilter, setRatingFilter] = useState<number | null>(null);

  const query = useInfiniteQuery({
    queryKey: ['reviews', productId, sort, ratingFilter],
    queryFn:  ({ pageParam = 1 }) =>
                reviewsApi.list({ productId, sort, rating: ratingFilter, page: pageParam }),
    getNextPageParam: (lastPage) => lastPage.hasMore ? lastPage.page + 1 : undefined,
  });

  return { ...query, sort, setSort, ratingFilter, setRatingFilter };
}

function ReviewSummary({ productId }) {
  const { data: summary } = useQuery({
    queryKey: ['review-summary', productId],
    queryFn:  () => reviewsApi.getSummary(productId), // { average, total, distribution: {5: 120, 4: 45, ...} }
  });

  if (!summary) return <ReviewSummarySkeleton />;

  return (
    <div className="review-summary">
      <div className="average-rating">
        <span className="rating-number">{summary.average.toFixed(1)}</span>
        <StarRating value={summary.average} />
        <span className="total-count">{summary.total} reviews</span>
      </div>
      <div className="rating-distribution">
        {[5, 4, 3, 2, 1].map(stars => (
          <div key={stars} className="distribution-row">
            <span>{stars} star</span>
            <div className="bar">
              <div
                className="bar-fill"
                style={{ width: `${(summary.distribution[stars] / summary.total) * 100}%` }}
              />
            </div>
            <span>{summary.distribution[stars]}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Key decision:** the review summary (average rating, distribution histogram) is fetched SEPARATELY from the paginated review list — it's a single, cacheable aggregate that doesn't need to be recomputed client-side from potentially thousands of review records, and it doesn't change when the user changes sort order or pagination.

---

## Step 6 — SEO: Structured Data and Server Rendering

```html
<!-- This page MUST be server-rendered (SSR/SSG) for SEO — a client-only
     SPA product page will rank poorly and won't generate rich search results -->

<script type="application/ld+json">
  {
    "@context": "https://schema.org/",
    "@type": "Product",
    "name": "Classic Running Shoe",
    "image": ["https://cdn.example.com/shoe-1.jpg"],
    "description": "...",
    "sku": "SHOE-123",
    "brand": { "@type": "Brand", "name": "ExampleBrand" },
    "offers": {
      "@type": "Offer",
      "url": "https://example.com/products/shoe-123",
      "priceCurrency": "USD",
      "price": "89.99",
      "availability": "https://schema.org/InStock"
    },
    "aggregateRating": {
      "@type": "AggregateRating",
      "ratingValue": "4.6",
      "reviewCount": "312"
    }
  }
</script>
```

```
SEO CHECKLIST:
☐ Server-rendered HTML contains the product name, price, and description
  (not just a loading spinner that JS fills in later)
☐ Product schema (JSON-LD) present and accurate — enables rich snippets
  in search results (star ratings, price, availability)
☐ Canonical URL set correctly (variant query params shouldn't create
  duplicate-content issues — use rel="canonical" pointing to the base URL)
☐ Meta description and Open Graph tags for social sharing previews
☐ Image alt text for all product images (accessibility + image search SEO)
```

---

## Performance Checklist

```
☐ Main product image has fetchPriority="high" and loading="eager"
☐ Thumbnail and gallery images lazy-load
☐ Price/stock display has reserved space to avoid CLS on variant change
☐ Reviews paginate rather than loading all at once
☐ Related products carousel lazy-loads images below the fold
☐ Critical CSS for above-the-fold content inlined (if not using a framework
  that handles this automatically)
☐ Variant data is small enough to embed in the initial HTML payload
  (avoid an extra round-trip just to populate the size/color selectors)
```

---

## Extension Ideas

```
- "Recently viewed products" using localStorage
- Size guide modal with measurements
- Wishlist with persisted state across sessions
- "Notify me when back in stock" for out-of-stock variants
- Compare products feature (side-by-side comparison table)
- Augmented reality "view in your space" for applicable products
- Live inventory countdown for high-demand items ("3 people viewing this")
```

---

## 🔗 Related Topics

- [`performance/`](../performance/) — Core Web Vitals optimization techniques used throughout
- [`browser-internals/10-ssr-csr-isr-streaming.md`](../browser-internals/10-ssr-csr-isr-streaming.md) — SSR rationale for SEO
- [`caching/`](../caching/) — Caching strategy for product/review data
- [`patterns/04-controlled-uncontrolled.md`](../patterns/04-controlled-uncontrolled.md) — Variant selector state pattern

---

<div align="center">

**Next:** [`projects/03-kanban-board.md`](./03-kanban-board.md) →

</div>
