# 10 — SSR, CSR, ISR & Streaming

> **"Every rendering strategy is a different answer to the same question: where and when do we run the code that turns data into HTML? The right answer depends on your content, your users, your infrastructure, and the tradeoffs you're willing to make."**

The rendering strategy you choose shapes everything: First Contentful Paint, Time to Interactive, SEO, infrastructure cost, developer experience, and how users experience your application. This document covers all four major strategies — Server-Side Rendering (SSR), Client-Side Rendering (CSR), Incremental Static Regeneration (ISR), and Streaming SSR — from first principles, with honest tradeoffs and the mental model for choosing between them.

---

## 📚 Table of Contents

1. [The Core Question — Where Does Rendering Happen?](#1-the-core-question--where-does-rendering-happen)
2. [CSR — Client-Side Rendering](#2-csr--client-side-rendering)
3. [SSR — Server-Side Rendering](#3-ssr--server-side-rendering)
4. [Static Site Generation (SSG)](#4-static-site-generation-ssg)
5. [ISR — Incremental Static Regeneration](#5-isr--incremental-static-regeneration)
6. [Streaming SSR](#6-streaming-ssr)
7. [Hydration — The Bridge Between SSR and CSR](#7-hydration--the-bridge-between-ssr-and-csr)
8. [Partial Hydration and Islands Architecture](#8-partial-hydration-and-islands-architecture)
9. [The Rendering Strategy Decision Framework](#9-the-rendering-strategy-decision-framework)
10. [Performance Metrics by Strategy](#10-performance-metrics-by-strategy)
11. [SEO Implications](#11-seo-implications)
12. [Infrastructure Tradeoffs](#12-infrastructure-tradeoffs)
13. [Rendering at the Edge](#13-rendering-at-the-edge)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. The Core Question — Where Does Rendering Happen?

Every web page is ultimately HTML. The question is: **where is that HTML generated?**

```
DATA + TEMPLATE = HTML

Options:
  A) Server generates HTML for every request (SSR)
  B) Build server generates HTML once (SSG/ISR)
  C) Browser generates HTML from JavaScript (CSR)
  D) Server generates HTML in chunks as data becomes ready (Streaming SSR)
  E) Mix: some on server, some on client (Partial Hydration / Islands)
```

### The Timeline That Matters

```
User perspective: time from URL to interactive page

Strategy comparison (rough, fast network):

CSR:   ──────────────────────────────────────► interactive
                    ↑ blank until JS loads + fetches data
                    ~3s on fast connection, 10s+ on slow

SSR:   ─────────────► interactive
              ↑ HTML with content on first byte
              ~800ms on fast connection

SSG:   ──────────────► interactive
               ↑ pre-built HTML, served from CDN
               ~300ms (no server processing)

Stream: ──────────────► interactive
               ↑ starts painting immediately
               as data becomes ready, progressively
```

---

## 2. CSR — Client-Side Rendering

In CSR, the server sends a minimal HTML shell. The browser downloads JavaScript, which fetches data and renders the UI entirely in the browser.

### How CSR Works

```
User requests: https://example.com/dashboard

Server sends:
  <!DOCTYPE html>
  <html>
  <head>
    <title>App</title>
    <script src="/app.bundle.js" defer></script>
  </head>
  <body>
    <div id="root"></div>  ← empty shell
  </body>
  </html>

Browser receives HTML (tiny, renders immediately):
  User sees: blank/loading screen

Browser downloads app.bundle.js (300KB, ~500ms on fast 3G):
  User still sees: blank/loading screen

JavaScript executes, React/Vue/Angular bootstraps:
  User sees: loading spinner (maybe)

JavaScript calls /api/dashboard (another 200ms):
  User still sees: loading spinner

Data arrives, components render:
  User sees: actual dashboard content
  Time: ~1.5-3s+ until useful content
```

### CSR Characteristics

```
Pros:
  ✦ Rich interactivity — full JavaScript runtime from first render
  ✦ Fast subsequent navigation — no server round trips for page changes
  ✦ Simple deployment — static files on a CDN
  ✦ Server cost — minimal (just static file serving)
  ✦ Developer experience — fast local development cycle
  ✦ Offline capability — service worker can serve cached shell

Cons:
  ✦ Slow First Contentful Paint (FCP) — blank until JS loads
  ✦ Slow Largest Contentful Paint (LCP) — content needs API call
  ✦ Poor SEO baseline — crawlers may not execute JavaScript
  ✦ Heavy JavaScript payload on initial load
  ✦ CPU-intensive on low-end devices (JS parsing + execution)
```

### When CSR Makes Sense

```
Good fit for CSR:
  ✓ Private applications (login required — SEO irrelevant)
  ✓ Highly interactive tools (dashboards, admin panels, editors)
  ✓ Real-time applications (live data, collaboration tools)
  ✓ PWAs with offline requirement
  ✓ Applications where users are on fast connections (B2B tools)

Poor fit for CSR:
  ✗ Public-facing pages that need SEO (blog, e-commerce, marketing)
  ✗ Content-heavy sites where first load matters
  ✗ Audiences on slow devices or networks
```

### CSR Performance Optimization

```javascript
// Code splitting: load only what's needed for current route
const Dashboard = React.lazy(() => import('./Dashboard'));
const Profile   = React.lazy(() => import('./Profile'));

// Prefetch likely next routes
<link rel="prefetch" href="/chunk-dashboard.js">

// Skeleton screens instead of spinners
function DashboardSkeleton() {
  return (
    <div className="skeleton">
      <div className="skeleton__header" />
      <div className="skeleton__content" />
    </div>
  );
}
```

---

## 3. SSR — Server-Side Rendering

In SSR, the server generates complete HTML on every request. The browser receives fully rendered HTML and can display content immediately.

### How SSR Works

```
User requests: https://example.com/product/123

Server:
  1. Receives request
  2. Fetches product data from database (20ms)
  3. Renders React/Vue component tree to HTML string (5ms)
  4. Returns complete HTML:

  <!DOCTYPE html>
  <html>
  <head><title>Widget Pro - Product</title></head>
  <body>
    <nav>...</nav>
    <main>
      <h1>Widget Pro</h1>
      <p>The best widget available.</p>
      <img src="/widget.jpg" alt="Widget Pro">
      <button>Add to Cart</button>
    </main>
  </body>
  </html>

  5. Browser receives HTML — instantly shows complete content
  6. JavaScript loads in background
  7. React "hydrates" — attaches event listeners to existing DOM
  8. Page becomes interactive (button clicks work)
```

### The SSR Timeline

```
t=0ms:    Request sent
t=200ms:  TTFB — first byte of HTML arrives (server rendered)
t=220ms:  FCP — user sees content (HTML drawn immediately)
t=800ms:  JavaScript finishes loading
t=850ms:  Hydration complete — interactive
```

vs CSR:

```
t=0ms:    Request sent
t=100ms:  TTFB — HTML shell arrives (empty div)
t=100ms:  HTML drawn — blank/loading screen visible
t=600ms:  JavaScript loaded
t=700ms:  API call completes
t=750ms:  FCP — content visible
t=750ms:  Interactive (already hydrated on first render)
```

### SSR Characteristics

```
Pros:
  ✦ Fast FCP/LCP — HTML arrives with content, no JS needed to see it
  ✦ SEO — crawlers receive full HTML content
  ✦ Works without JavaScript — progressive enhancement possible
  ✦ Social sharing — OG tags available from server

Cons:
  ✦ Server cost — compute on every request
  ✦ TTFB — server must fetch data before responding
  ✦ Time to Interactive — HTML visible but not interactive until JS hydrates
  ✦ Infrastructure — needs a server (not just a CDN)
  ✦ Cache complexity — personalised content harder to cache
  ✦ Data fetching on server — different mental model than client fetching
```

### The TTFB Problem in SSR

```
SSR bottleneck: server must complete all data fetching BEFORE sending HTML

Waterfall on server (sequential):
  Receive request → fetch user data (50ms) → fetch product data (80ms)
  → fetch recommendations (100ms) → render HTML → send first byte

  TTFB: 230ms+ just for data fetching

Fix 1: Parallel data fetching
  → fetch user + product + recommendations simultaneously
  → TTFB: 100ms (slowest fetch, not sum)

Fix 2: Streaming SSR (covered later)
  → Start sending HTML immediately, stream data as it arrives
```

### Node.js SSR Example

```javascript
// Express SSR with React
import express from "express";
import { renderToString } from "react-dom/server";
import App from "./App";

const app = express();

app.get("/product/:id", async (req, res) => {
  try {
    // Parallel data fetching
    const [product, user, recommendations] = await Promise.all([
      fetchProduct(req.params.id),
      fetchUser(req.session.userId),
      fetchRecommendations(req.params.id),
    ]);

    // Render to HTML string
    const html = renderToString(
      <App product={product} user={user} recommendations={recommendations} />,
    );

    // Send complete page
    res.send(`
      <!DOCTYPE html>
      <html>
        <head>
          <title>${product.name}</title>
          <meta name="description" content="${product.description}">
        </head>
        <body>
          <div id="root">${html}</div>
          <script>
            window.__INITIAL_DATA__ = ${JSON.stringify({
              product,
              user,
              recommendations,
            })};
          </script>
          <script src="/client.js" defer></script>
        </body>
      </html>
    `);
  } catch (err) {
    res.status(500).send("Error");
  }
});
```

---

## 4. Static Site Generation (SSG)

SSG generates HTML at **build time** — not at request time. Pages are pre-built and served as static files from a CDN.

### How SSG Works

```
BUILD TIME (happens once, before deployment):
  1. Framework fetches all data from CMS/API
  2. Generates static HTML for every page
  3. Outputs: index.html, /blog/post-1.html, /products/widget.html, etc.
  4. Deploys to CDN

REQUEST TIME:
  User requests /blog/post-1
  CDN serves pre-built HTML instantly (< 50ms)
  No server rendering
  No database query
  No compute
```

### SSG Characteristics

```
Pros:
  ✦ Fastest possible delivery — pre-built HTML from CDN
  ✦ Zero server compute at request time
  ✦ Minimal infrastructure cost
  ✦ Perfect cache-ability — immutable files
  ✦ Excellent SEO — complete HTML immediately
  ✦ Resilient — no server to go down

Cons:
  ✦ Stale content — rebuilt only on deploy
  ✦ Build time scales with page count (1000 pages = long builds)
  ✦ Not suitable for personalized content
  ✦ Not suitable for highly dynamic data
  ✦ Redeploy required to update content
```

### When SSG Makes Sense

```
Excellent fit:
  ✓ Marketing sites, landing pages
  ✓ Documentation
  ✓ Blogs, news articles
  ✓ Product catalogs (relatively static)
  ✓ Any content that changes infrequently

Poor fit:
  ✗ Real-time data (prices, inventory)
  ✗ User-specific content
  ✗ Frequently updated content (news at large scale)
  ✗ Very large page counts (millions of pages → hours of build time)
```

---

## 5. ISR — Incremental Static Regeneration

ISR (pioneered by Next.js) combines the benefits of SSG (CDN-served static HTML) with the ability to update content without a full rebuild.

### How ISR Works

```
First request to /products/widget:
  → Page not yet generated (or stale)
  → Server generates HTML fresh (like SSR)
  → Response served to user
  → Response cached (stored as static HTML)

All subsequent requests (within revalidate period):
  → Served from cache (like SSG — instant from CDN)
  → No server work

After revalidate period (e.g., 60 seconds):
  First request after expiry:
    → Served from stale cache immediately (user gets fast response)
    → Server regenerates page in background
    → Next request gets fresh cached version
```

### ISR vs SSG vs SSR

```
SSG:
  Build time → static HTML
  Update requires: full site rebuild + deploy

ISR:
  First request → generate + cache
  After revalidate period → regenerate in background
  Update requires: wait for revalidation (no deploy needed)

SSR:
  Every request → generate HTML
  Always fresh
  Always requires server compute
```

### ISR Implementation (Next.js)

```javascript
// pages/products/[id].js (Next.js Pages Router)
export async function getStaticProps({ params }) {
  const product = await fetchProduct(params.id);

  return {
    props: { product },
    revalidate: 60, // regenerate at most every 60 seconds
  };
}

export async function getStaticPaths() {
  // Pre-build popular products at build time
  const topProducts = await fetchTopProducts(100);
  return {
    paths: topProducts.map((p) => ({ params: { id: p.id } })),
    fallback: "blocking", // generate on-demand for other products
    // fallback: true        → serve fallback page while generating
    // fallback: 'blocking'  → wait for generation, no fallback
    // fallback: false       → 404 for unknown paths
  };
}

export default function ProductPage({ product }) {
  return <Product product={product} />;
}
```

### On-Demand ISR

```javascript
// Trigger revalidation programmatically (Next.js 12.2+)
// Use when content changes in CMS — don't wait for revalidate timer

// API route: /api/revalidate
export default async function handler(req, res) {
  if (req.query.secret !== process.env.REVALIDATION_SECRET) {
    return res.status(401).json({ message: "Invalid token" });
  }

  try {
    // Revalidate specific page(s)
    await res.revalidate(`/products/${req.query.id}`);
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send("Error revalidating");
  }
}

// CMS webhook calls this endpoint when content is updated
// Specific page gets regenerated immediately — no need to wait for timer
```

---

## 6. Streaming SSR

Streaming SSR sends HTML in chunks as it becomes ready, rather than waiting for the entire page to be fully rendered before sending the first byte.

### The Problem with Traditional SSR

```
Traditional SSR waterfall:

Server:
  [fetch user: 50ms] → [fetch products: 80ms] → [fetch cart: 30ms]
                                                  ↓ (all three done)
  [render entire page HTML] → send first byte

TTFB: 160ms (sum of all fetches, if sequential)
FCP: 200ms after request
User waits: 160ms+ for anything to appear
```

### How Streaming SSR Works

```
Streaming SSR:

Server starts responding immediately:
  t=0ms: Send HTML head + shell + critical above-fold HTML
  t=10ms: User sees: header, navigation, page structure

  While browser renders partial HTML:
  Server fetches data concurrently

  t=50ms: User data ready → stream HTML for sidebar
  t=80ms: Products ready → stream HTML for product grid
  t=100ms: Cart ready → stream HTML for cart widget

  t=100ms: Page complete, all HTML delivered

TTFB: ~5ms (shell sent immediately)
FCP: ~15ms (critical content visible)
Largest content fully loaded: ~100ms
```

### React Streaming with Suspense

```javascript
// Server (Node.js with React 18)
import { renderToPipeableStream } from "react-dom/server";

app.get("/dashboard", (req, res) => {
  const { pipe } = renderToPipeableStream(
    <App>
      {/* Static header: sent immediately */}
      <Header />

      {/* Suspense boundary: deferred until data is ready */}
      <Suspense fallback={<ProductsSkeleton />}>
        <ProductGrid /> {/* Streams when ready */}
      </Suspense>

      <Suspense fallback={<CartSkeleton />}>
        <CartWidget /> {/* Streams when ready */}
      </Suspense>
    </App>,
    {
      onShellReady() {
        res.statusCode = 200;
        res.setHeader("Content-Type", "text/html");
        pipe(res); // Start streaming immediately when shell is ready
      },
      onAllReady() {
        // All content is ready — streaming complete
      },
      onError(err) {
        console.error(err);
      },
    },
  );
});
```

```javascript
// Client component with async data (React Server Components)
// This component's data is fetched on the server, HTML streamed to client

async function ProductGrid() {
  const products = await fetchProducts(); // Await on server
  return (
    <div className="product-grid">
      {products.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

### Streaming HTML Wire Format

```
What the browser receives over time:

Chunk 1 (immediate):
  <!DOCTYPE html><html><head>...</head><body>
  <div id="root">
    <header>...</header>
    <main>
      <!--$?--><template id="B:0"></template><!--/$-->
      <!--$?--><template id="B:1"></template><!--/$-->
    </main>

Chunk 2 (when ProductGrid data ready, ~80ms):
  <div hidden id="S:0">
    <div class="product-grid">
      <div class="product-card">Widget Pro</div>
      ...
    </div>
  </div>
  <script>$RC("B:0","S:0")</script>
  <!-- $RC = React replace content: swaps template with actual content -->

Chunk 3 (when CartWidget data ready, ~100ms):
  <div hidden id="S:1">
    <div class="cart">3 items</div>
  </div>
  <script>$RC("B:1","S:1")</script>

</body></html>
```

### Streaming Characteristics

```
Pros:
  ✦ TTFB as low as SSG — shell sent immediately
  ✦ Progressive rendering — users see content as it arrives
  ✦ No waterfall — server fetches all data in parallel
  ✦ Better LCP than traditional SSR
  ✦ Works well with slow data sources (one slow query doesn't block others)

Cons:
  ✦ Requires HTTP/1.1 chunked transfer or HTTP/2 (not HTTP/1.0)
  ✦ More complex than traditional SSR
  ✦ CDN caching is complex (can't cache partial streams)
  ✦ Framework support required (React 18+, etc.)
  ✦ Debugging is harder (streaming responses)
```

---

## 7. Hydration — The Bridge Between SSR and CSR

Hydration is the process of attaching JavaScript event listeners and state to server-rendered HTML. It's necessary for SSR/SSG pages to become interactive.

### What Hydration Does

```
After SSR sends HTML to browser:

Step 1: Browser receives and paints HTML
  User sees: fully rendered page (static)
  User tries to click button: nothing happens (no JS yet)

Step 2: JavaScript bundle downloads and executes
  React/Vue walks the existing DOM
  Matches DOM nodes to component tree
  Attaches event listeners
  Initializes component state

Step 3: Hydration complete
  User clicks button: works ✓
  State management active ✓
  Client-side routing works ✓
```

### Hydration Mismatch — A Common Bug

```javascript
// ❌ Hydration mismatch: server and client render different HTML
function DashboardHeader() {
  // This produces different output on server vs client!
  return <div>Current time: {new Date().toLocaleString()}</div>;
  // Server: "Current time: 1/15/2024, 10:30:00 AM"
  // Client (100ms later): "Current time: 1/15/2024, 10:30:01 AM"
  // React throws warning, attempts to recover
}

// ❌ Another mismatch: accessing browser APIs on server
function WindowWidth() {
  return <div>Width: {window.innerWidth}px</div>;
  // Server: window is undefined → throws!
}

// ✅ Fix: use useEffect for browser-only code
function WindowWidth() {
  const [width, setWidth] = React.useState(null);
  React.useEffect(() => {
    setWidth(window.innerWidth);
  }, []);
  return <div>Width: {width !== null ? `${width}px` : "Loading..."}</div>;
}
```

### The Hydration Performance Problem

```
SSR + Hydration timeline:

t=0ms:   Request
t=200ms: TTFB — HTML arrives
t=220ms: FCP — user sees content
            ↓
t=700ms: JS downloaded
t=800ms: Hydration starts
            During hydration: main thread is BUSY
            Clicks are queued but not processed
            Page looks interactive but isn't
t=900ms: Hydration complete — actually interactive

"Time to Interactive" gap: 700ms of non-interactive visual content
User frustration: clicks button, nothing happens
```

### Measuring Hydration Performance

```javascript
// React 18 provides a transition API for deferred hydration
import { startTransition } from "react";

// Wrap hydration in a transition — lower priority
startTransition(() => {
  hydrateRoot(document.getElementById("root"), <App />);
});
// Input handling and higher-priority UI remain responsive during hydration
```

---

## 8. Partial Hydration and Islands Architecture

Instead of hydrating the entire page (full hydration), partial hydration only hydrates the **interactive parts** (islands). Static parts stay as plain HTML with no JavaScript overhead.

### The Problem with Full Hydration

```
Traditional SSR page:

<nav>       ← static, just links, no interactivity needed
  <a href="/">Home</a>
  <a href="/products">Products</a>
</nav>

<article>   ← static content, just text, no interactivity needed
  <h1>About Us</h1>
  <p>Long article content...</p>
</article>

<div id="cart-widget">  ← INTERACTIVE: needs state, click handlers
  Cart: 3 items
</div>

<form id="contact">    ← INTERACTIVE: needs validation, submission
  ...
</form>

Full hydration: hydrate EVERYTHING including nav and article
→ React.hydrate() walks entire DOM tree
→ Sends entire React component tree to browser
→ Parses and executes even for static content

Waste: hydrating nav and article = pure overhead, no user benefit
```

### Islands Architecture

```
Islands approach:
  Static parts: pure HTML/CSS — no JavaScript
  Interactive parts (islands): independently hydrated

  <nav>       ← static HTML, zero JavaScript
  <article>   ← static HTML, zero JavaScript
  <CartWidget component="react">   ← island: hydrated independently
  <ContactForm component="react">  ← island: hydrated independently

Benefits:
  - Total JS sent to browser: only CartWidget + ContactForm bundles
  - No React runtime for static content
  - Islands hydrate independently and lazily
  - Extremely fast TTI for pages with mostly static content
```

### Islands Implementation (Astro)

```astro
---
// index.astro
const products = await fetchProducts();
---

<!-- Static: no JavaScript sent to browser -->
<nav>
  <a href="/">Home</a>
  <a href="/products">Products</a>
</nav>

<!-- Static: product list rendered to HTML, no JS -->
<ul>
  {products.map(p => <li>{p.name}</li>)}
</ul>

<!-- Island: React component, hydrated on load -->
<CartWidget client:load />

<!-- Island: hydrated only when visible -->
<NewsletterSignup client:visible />

<!-- Island: hydrated on first interaction -->
<VideoPlayer client:idle />
```

### Hydration Directives (Astro)

```
client:load     — hydrate immediately on page load
client:idle     — hydrate when browser is idle (requestIdleCallback)
client:visible  — hydrate when element enters viewport (IntersectionObserver)
client:media="(max-width: 768px)"  — hydrate when media query matches
client:only="react"  — only render on client, skip server render
```

---

## 9. The Rendering Strategy Decision Framework

```
Decision tree for choosing a rendering strategy:

Is the content the same for all users?
  YES → Is it truly static (changes < weekly)?
            YES → SSG (best performance, lowest cost)
            NO  → ISR (static performance + periodic freshness)
  NO  → Is it personalized per user?
            YES → SSR or CSR
                   Does it need SEO?
                     YES → SSR
                     NO  → CSR
            Mixed → Hybrid (SSG/SSR shell + CSR for personalized parts)
```

### Decision Matrix

| Factor          |    CSR     |       SSR        |       SSG        |       ISR        |  Streaming SSR  |
| --------------- | :--------: | :--------------: | :--------------: | :--------------: | :-------------: |
| FCP / LCP       |  ❌ Slow   |     ✅ Good      |    ✅✅ Best     |    ✅✅ Best     |    ✅✅ Best    |
| TTI             |  ✅ Fast   | ⚠️ Hydration gap | ⚠️ Hydration gap | ⚠️ Hydration gap |     ✅ Good     |
| SEO             |  ⚠️ Risky  |     ✅ Good      |    ✅✅ Best     |    ✅✅ Best     |     ✅ Good     |
| Dynamic content |    ✅✅    |       ✅✅       |        ❌        |    ⚠️ Delayed    |      ✅✅       |
| Personalization |    ✅✅    |        ✅        |        ❌        |        ❌        |       ✅        |
| Server cost     |  ✅✅ Low  |  ⚠️ Per request  | ✅✅ Build only  |   ✅ Amortized   | ⚠️ Per request  |
| Cacheability    |    ✅✅    |    ⚠️ Complex    |   ✅✅ Perfect   |     ✅ Good      |   ⚠️ Partial    |
| Infrastructure  | Simple CDN | Server required  |     CDN only     |   Server + CDN   | Server required |

---

## 10. Performance Metrics by Strategy

### FCP (First Contentful Paint)

```
SSG:       50–300ms   (CDN serves pre-built HTML)
Streaming: 100–500ms  (shell sent immediately)
SSR:       200–800ms  (server renders before responding)
CSR:       800–3000ms (must download + execute JS first)
```

### LCP (Largest Contentful Paint)

```
SSG:       200–600ms   (image preloaded, HTML from CDN)
Streaming: 300–1000ms  (content streams in progressively)
SSR:       400–1200ms  (full HTML must be generated)
CSR:       1500–5000ms (JS + API call + render)
```

### Time to Interactive (TTI)

```
CSR:       800–2000ms  (interactive on first render — no hydration)
Streaming: 1000–2500ms (progressive hydration)
SSR:       1200–3000ms (full hydration after HTML loads)
SSG:       1200–3000ms (same as SSR — hydration still needed)
```

### Note on Hydration Gap

All SSR/SSG strategies suffer from the "hydration gap" — time between FCP (content visible) and TTI (actually interactive):

```
Hydration gap strategies to reduce:
  1. Minimize JS bundle size — less to parse and execute
  2. Use Islands architecture — only hydrate what needs it
  3. React 18 concurrent features — lower priority hydration
  4. Progressive hydration — hydrate above fold first
  5. Resume (Qwik) — serialize server state, no re-execution
```

---

## 11. SEO Implications

### Googlebot and JavaScript Rendering

```
Googlebot has two crawl queues:

1. Immediate crawl:
   Renders HTML as-is (no JavaScript execution)
   Fast, determines initial indexing

2. Deferred second wave:
   Waits for JavaScript to execute
   Can take hours to days after initial crawl
   Used for SPA/CSR content

For SSR/SSG:
  Googlebot gets full HTML immediately → Indexed in wave 1 → Fast SEO

For CSR:
  Wave 1: sees empty div → no content indexed
  Wave 2: JavaScript executes → content indexed (eventually)
  Risk: core content may not be indexed in time
```

### Social Sharing (OG Tags)

```
Social media crawlers (Facebook, Twitter, LinkedIn, Slack):
  → Do NOT execute JavaScript
  → Read HTML source only
  → If OG tags are in initial HTML: previews work
  → If OG tags require JavaScript: no preview

SSR/SSG: OG tags in HTML → ✅ Social previews work
CSR: OG tags added by JS → ❌ Social previews broken
```

### Dynamic OG Tags with SSR

```javascript
// SSR: generate OG tags from data for each page
app.get("/product/:id", async (req, res) => {
  const product = await fetchProduct(req.params.id);
  res.send(`
    <!DOCTYPE html>
    <html>
    <head>
      <title>${product.name}</title>
      <meta name="description" content="${product.description}">
      <meta property="og:title" content="${product.name}">
      <meta property="og:description" content="${product.description}">
      <meta property="og:image" content="${product.imageUrl}">
      <meta property="og:url" content="https://example.com/product/${product.id}">
    </head>
    ...
  `);
});
```

---

## 12. Infrastructure Tradeoffs

### CSR Infrastructure

```
Requirements:
  - Static file hosting (CDN)
  - API server(s)

Cost: low (CDN is cheap; API servers are standard)
Scaling: trivial — CDN scales automatically
Complexity: simple — just files + API

Tools: GitHub Pages, Netlify, Vercel (static), S3 + CloudFront
```

### SSR Infrastructure

```
Requirements:
  - Node.js server (or equivalent)
  - Load balancer (for scale)
  - Session/memory management

Cost: moderate — compute per request
Scaling: horizontal — more server instances under load
Complexity: moderate — server management, cold starts

Tools: Vercel (serverless), Netlify Functions, AWS Lambda,
       dedicated Node.js servers
```

### SSG/ISR Infrastructure

```
SSG:
  - CDN for static files
  - Build server (CI/CD)
  - No request-time server
  Cost: very low

ISR:
  - CDN for cached pages
  - Server for on-demand generation
  - CDN cache invalidation mechanism
  Cost: moderate (server is used sparingly)

Tools: Vercel, Netlify, Cloudflare Pages
```

---

## 13. Rendering at the Edge

Edge computing runs code at CDN nodes worldwide — milliseconds from any user, not in a single centralized server.

### Edge SSR

```
Traditional SSR:
  User (Tokyo) → Server (US East) → Response
  Latency: ~150ms just for TCP round trip

Edge SSR:
  User (Tokyo) → Edge node (Tokyo) → Response
  Latency: ~5ms TCP round trip

Code runs at nearest CDN node using a V8 isolate (not a full Node.js process)
Limitations: no file system, limited compute, restricted APIs
```

### Edge Worker Example (Cloudflare Workers)

```javascript
// Runs at the CDN edge — sub-millisecond cold start
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Serve from cache if available
    const cached = await caches.default.match(request);
    if (cached) return cached;

    // Generate response at edge
    const data = await fetchFromDB(url.pathname, env);
    const html = renderTemplate(data);

    const response = new Response(html, {
      headers: {
        "Content-Type": "text/html",
        "Cache-Control": "public, max-age=300, stale-while-revalidate=86400",
      },
    });

    // Cache at edge
    event.waitUntil(caches.default.put(request, response.clone()));

    return response;
  },
};
```

### Edge Rendering Limitations

```
What works at the edge:
  ✓ Template rendering
  ✓ Simple data fetching (via fetch API)
  ✓ Caching
  ✓ Authentication/authorization
  ✓ A/B testing
  ✓ Geolocation-based content

What doesn't work at the edge:
  ✗ Heavy database queries (latency to central DB)
  ✗ File system access
  ✗ Most npm packages (incompatible APIs)
  ✗ Long-running compute (time limits ~50ms)
```

---

## 14. Good Practices

### ✅ Match rendering strategy to content type

```javascript
// Next.js: different strategies per page

// SSG: blog post (static content)
export async function getStaticProps({ params }) {
  return { props: { post: await fetchPost(params.slug) }, revalidate: 3600 };
}

// SSR: user dashboard (personalized)
export async function getServerSideProps(context) {
  const user = await getUser(context.req.session.userId);
  return { props: { user } };
}

// CSR: live data widget (no SSR needed — data always changes)
function LiveWidget() {
  const data = useRealtimeData("/api/live"); // client-side only
  return <Chart data={data} />;
}
```

### ✅ Use Suspense boundaries for streaming

```javascript
// ✅ Wrap slow-loading sections in Suspense
<Suspense fallback={<Skeleton />}>
  <SlowDataComponent />  {/* streams when ready */}
</Suspense>

// Multiple independent Suspense boundaries = maximum parallelism
<Suspense fallback={<NavSkeleton />}>
  <Navigation />
</Suspense>
<Suspense fallback={<ContentSkeleton />}>
  <MainContent />
</Suspense>
```

### ✅ Avoid hydration mismatches

```javascript
// ✅ Guard browser-only code
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);

if (!mounted) return <StaticPlaceholder />;
return <BrowserOnlyComponent />;
```

### ✅ Use Islands for mostly-static pages

```
If your page is 90% static content + 10% interactive widgets:
  → Islands architecture: only send JS for the 10%
  → Massive TTI and bundle size improvements
```

---

## 15. Bad Practices

### ❌ CSR for SEO-critical public pages

```
❌ Using pure CSR for:
  - Product pages (need SEO, social sharing)
  - Blog articles (need search indexing)
  - Landing pages (Google needs to crawl them)

These require at minimum SSR or SSG.
```

### ❌ SSR without caching

```javascript
// ❌ Generating the same HTML for every request — expensive
app.get("/about", (req, res) => {
  const html = renderToString(<AboutPage />);
  res.send(html);
  // About page never changes — should be cached or SSG'd
});

// ✅ Cache SSR responses
app.get("/about", async (req, res) => {
  const cached = await redis.get("page:about");
  if (cached) return res.send(cached);

  const html = renderToString(<AboutPage />);
  await redis.setex("page:about", 3600, html); // cache 1 hour
  res.send(html);
});
```

### ❌ Full hydration for mostly-static pages

```javascript
// ❌ Shipping React runtime + hydrating entire page for a blog article
hydrateRoot(document, <EntireBlogApp />);
// 98% of page is static text — no interactivity needed
// React runtime: 40KB, hydration cost: wasted

// ✅ Only hydrate interactive parts (comments, share button)
hydrateRoot(document.getElementById("comments"), <Comments />);
hydrateRoot(document.getElementById("share"), <ShareButton />);
```

### ❌ Ignoring the hydration gap

```
❌ Presenting SSR/SSG as "instant interactive" — it's not
The page looks interactive (HTML rendered) but isn't
(event handlers not attached yet)

Always measure TTI, not just FCP
Consider progressive hydration or Islands to minimize the gap
```

---

## 16. Common Mistakes

### Mistake 1 — SSR is not automatically SEO-friendly

```
SSR generates HTML on the server — good.
But: if that HTML doesn't include meaningful content or
proper meta tags, SEO is still poor.

Required for good SEO:
  ✓ Full page content in initial HTML
  ✓ Correct title tags
  ✓ Meta description tags
  ✓ Structured data (JSON-LD)
  ✓ Canonical URLs
  ✓ Open Graph tags
  ✓ Fast TTFB (< 200ms) — Google considers load speed
```

### Mistake 2 — ISR revalidate doesn't mean "refresh every N seconds"

```
ISR revalidate: 60 (60 seconds):

NOT: "rebuild every 60 seconds"
YES: "if a request comes in after 60 seconds from last build,
      serve the stale version immediately AND trigger a background rebuild"

Implication:
  - Page may serve stale content until the next request after expiry
  - If nobody visits for 24 hours, the page is 24 hours stale
  - On-demand revalidation is better for time-sensitive content
```

### Mistake 3 — Sending initial data twice with SSR

```javascript
// ❌ Data duplicated: in HTML and in JS bundle
const html = renderToString(<App products={products} />);
res.send(`
  ${html}
  <script>
    window.__INITIAL_DATA__ = ${JSON.stringify(products)};
  </script>
`);
// Products appear in both the rendered HTML AND the JSON
// If products is 50KB: 100KB total instead of 50KB

// ✅ Server Components (React) avoid this:
// Server components never sent to client — no JSON serialization needed
// Data stays on server, only HTML is sent
```

### Mistake 4 — Hydrating without code splitting

```javascript
// ❌ Entire app bundle required before hydration can start
import App from "./App"; // App includes all routes, all components
hydrateRoot(document, <App />);
// Must download + parse everything before ANY interactivity

// ✅ Selective/lazy hydration
const HeavyDashboard = React.lazy(() => import("./Dashboard"));
// Dashboard only hydrated when navigated to
// Initial page hydrates faster
```

---

## 17. Interview-Level Explanation

> **"What's the difference between SSR, CSR, SSG, ISR, and Streaming SSR? When would you use each?"**

**Strong answer:**

> "These are all different answers to 'where and when does HTML get generated?'
>
> CSR — Client-Side Rendering — sends a minimal HTML shell to the browser, then JavaScript fetches data and renders everything client-side. It's fast to develop and great for private apps like dashboards, but has poor FCP and LCP because the user sees a blank screen until JavaScript loads and fetches data. Not suitable for SEO-critical public pages.
>
> SSR — Server-Side Rendering — generates complete HTML on the server for every request. The browser gets fully rendered content immediately, so FCP and LCP are excellent. The downside is there's still a hydration gap — the page looks interactive before JavaScript attaches event listeners. SSR requires server infrastructure and compute per request.
>
> SSG — Static Site Generation — builds all pages as HTML at deploy time. They're served from a CDN with zero server compute, making them fastest for repeat visitors and trivially cacheable. The limitation is staleness — content is only updated on redeploy.
>
> ISR — Incremental Static Regeneration — solves SSG's staleness problem. Pages are cached statically but can be regenerated in the background after a configurable time period. Users always get fast responses from cache, and content can stay fresh without rebuilding the entire site.
>
> Streaming SSR — sends HTML in chunks as data becomes ready rather than waiting for the full page. The browser can start painting immediately, and slow data sources don't block fast ones. It achieves SSG-like TTFB with SSR-like freshness.
>
> The decision: use SSG for truly static content (docs, marketing), ISR for semi-static content (product pages, blog), SSR for personalized or highly dynamic content that needs SEO, and CSR for private apps that don't need SEO. Streaming SSR is the best default for SSR use cases in modern React. For pages that are mostly static with a few interactive widgets, Islands architecture hydrates only those widgets, dramatically reducing JavaScript sent to the browser."

---

## 18. Exercises

### Exercise 1 — Choose a strategy

For each application, recommend a rendering strategy and justify it:

```
a) An e-commerce product catalog with 50,000 products
   - Products change price ~once per day
   - Product descriptions rarely change
   - Must rank on Google

b) A real-time analytics dashboard
   - Data refreshes every 5 seconds
   - Only accessible after login
   - No SEO needed

c) A company blog
   - Posts updated 2-3 times per week
   - Must be discoverable on Google
   - No personalization

d) A social media feed
   - Unique per user
   - Must load fast
   - Not indexed by Google

e) A checkout flow
   - Cart data is per-user
   - Payment pages must be secure
   - Not indexed by Google
```

<details>
<summary>Answers</summary>

```
a) E-commerce product catalog:
   Strategy: ISR with revalidate = 86400 (1 day)
   Justification:
   - 50,000 products: SSG at build time is feasible but would be slow builds
   - Products change ~daily: ISR regenerates in background as needed
   - On-demand revalidation for price changes (webhook from pricing service)
   - CDN delivery: excellent FCP/LCP
   - Full HTML: excellent SEO
   - Price accuracy: on-demand ISR revalidation keeps prices current

b) Real-time analytics dashboard:
   Strategy: CSR
   Justification:
   - Private (login required): SEO irrelevant
   - Data changes every 5s: SSR/SSG would always be stale
   - CSR enables WebSocket/polling for live data
   - No public content: blank initial HTML is fine
   - Dashboard users expect loading states

c) Company blog:
   Strategy: SSG with on-demand ISR
   Justification:
   - Infrequent updates (2-3x/week): SSG is ideal
   - Zero compute at request time: CDN delivery
   - Excellent SEO: complete HTML
   - On-demand ISR: when author publishes, webhook triggers regeneration
   - New posts available within seconds, not minutes

d) Social media feed:
   Strategy: SSR for initial page + CSR for live updates
   Justification:
   - Personalized: SSR generates initial feed per-user
   - Fast FCP: user sees their feed immediately (no blank screen)
   - Live updates: CSR + WebSocket for new posts
   - Not SEO-indexed: personalized content doesn't need Google

e) Checkout flow:
   Strategy: CSR or SSR (security-first)
   Justification:
   - Not SEO-indexed: no Google requirement
   - Cart is per-user: can't be SSG or ISR
   - CSR works: no need for SSR (user expects loading states in checkout)
   - Security note: payment processing always server-side regardless
   - If fast checkout UX needed: SSR for initial cart state
```

</details>

---

### Exercise 2 — Debug the hydration mismatch

This component works in development but throws hydration warnings in production. Find and fix the issue.

```javascript
function BlogPost({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <time>{new Date(post.publishedAt).toLocaleDateString()}</time>
      <p className={Math.random() > 0.5 ? "highlight" : "normal"}>
        {post.content}
      </p>
      <div className="meta">
        Views: {localStorage.getItem(`views:${post.id}`) ?? 0}
      </div>
    </article>
  );
}
```

<details>
<summary>Solution</summary>

```javascript
// Three hydration issues:

// 1. toLocaleDateString() — locale-dependent, may differ between
//    server timezone and client timezone
// 2. Math.random() — produces different values on server vs client
// 3. localStorage — not available on server (throws ReferenceError)

// Fixed version:
function BlogPost({ post }) {
  // Fix 1: Use suppressHydrationWarning for inherently dynamic content
  // Or: use consistent date formatting (not locale-dependent)
  const formattedDate = new Intl.DateTimeFormat("en-US", {
    timeZone: "UTC", // consistent between server and client
    year: "numeric",
    month: "long",
    day: "numeric",
  }).format(new Date(post.publishedAt));

  // Fix 2: Don't use random for class determination — use data-based logic
  const className = post.featured ? "highlight" : "normal";

  // Fix 3: Browser-only code in useEffect
  const [views, setViews] = React.useState(0);
  React.useEffect(() => {
    const stored = localStorage.getItem(`views:${post.id}`);
    setViews(stored ? parseInt(stored, 10) : 0);
  }, [post.id]);

  return (
    <article>
      <h1>{post.title}</h1>
      <time dateTime={post.publishedAt}>{formattedDate}</time>
      <p className={className}>{post.content}</p>
      <div className="meta">Views: {views}</div>
    </article>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/08-critical-rendering-path.md`](./08-critical-rendering-path.md) — CRP affected by rendering strategy
- [`browser-internals/09-browser-caching.md`](./09-browser-caching.md) — Caching strategies for each approach
- [`system-design/01-large-scale-architecture.md`](../system-design/01-large-scale-architecture.md) — Architecture decisions including rendering
- [`performance/02-virtualization-windowing.md`](../performance/02-virtualization-windowing.md) — Client-side rendering performance

---

<div align="center">

**`browser-internals/` section complete!** 🎉

All 10 browser-internals files are done.

**Next section:** [`performance/`](../performance/) →

</div>
