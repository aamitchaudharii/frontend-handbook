# 05 — Hydration Patterns

> **"Hydration is the original sin of SSR. You server-render HTML so the user sees content immediately — then you make them wait while JavaScript reprocesses that same content to make it interactive. The page looks done but isn't. Hydration patterns are all the creative ways people have tried to pay that debt more efficiently."**

Hydration is the process of attaching JavaScript interactivity to server-rendered HTML. When the browser receives SSR HTML, it displays it immediately — but clicking a button does nothing until the JavaScript bundle downloads, parses, executes, and React "hydrates" the DOM by walking it and attaching event listeners. This hydration gap — the period when content is visible but not interactive — is one of the most significant UX problems in modern web development. This document covers hydration mechanics, hydration strategies that reduce or eliminate the gap, and the tradeoffs of each approach.

---

## 📚 Table of Contents

1. [What Hydration Is and Why It Matters](#1-what-hydration-is-and-why-it-matters)
2. [The Hydration Gap](#2-the-hydration-gap)
3. [Full Hydration — The Classic Approach](#3-full-hydration--the-classic-approach)
4. [Progressive Hydration](#4-progressive-hydration)
5. [Partial Hydration and Islands Architecture](#5-partial-hydration-and-islands-architecture)
6. [Lazy / On-Demand Hydration](#6-lazy--on-demand-hydration)
7. [Streaming SSR and Selective Hydration](#7-streaming-ssr-and-selective-hydration)
8. [Resumability — Hydration Without Replay](#8-resumability--hydration-without-replay)
9. [Hydration Mismatches](#9-hydration-mismatches)
10. [React Server Components — Beyond Hydration](#10-react-server-components--beyond-hydration)
11. [Measuring Hydration Performance](#11-measuring-hydration-performance)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. What Hydration Is and Why It Matters

### The Two-Pass Rendering Problem

```
PASS 1 (Server-Side Rendering):
  Server renders component tree → HTML string
  Browser receives HTML → displays immediately (FCP: ~200ms)
  User sees: full page content
  User tries to click: nothing happens

  At this point:
    ✅ Content visible
    ❌ No event listeners
    ❌ No React state
    ❌ No client-side routing

PASS 2 (Hydration — Client Side):
  JavaScript bundle downloads (~500KB, ~500ms on fast 3G)
  Bundle parses and executes (~300ms on low-end device)
  React walks the DOM, matches virtual DOM to real DOM
  React attaches event listeners → page finally interactive

  TTI: ~1,000ms after FCP

  Hydration gap: 800ms where content looks interactive but isn't
```

### Why This Matters for Real Users

```
User experience during hydration gap:
  Click button: nothing happens (event listener not attached yet)
  Type in search: no response
  Click link: may or may not work (depends on whether hydrated yet)

User perception:
  Page appeared fast (SSR FCP) but "broke" immediately
  Users retry clicks, assume page is broken
  Rage clicks, frustration, abandonment

Core Web Vital impact:
  FCP: fast (SSR benefits)
  TTI: slow (large JS bundle + hydration time)
  CLS: risk during hydration if positions change
  INP: degraded during hydration if event handlers queue
```

---

## 2. The Hydration Gap

```
TIMELINE:

0ms:      User navigates
200ms:    TTFB → HTML arrives
220ms:    FCP → content visible (SSR HTML painted)
                ↓ HYDRATION GAP BEGINS
220ms:    JavaScript bundle request dispatched
700ms:    Bundle downloaded (500KB at ~1.25MB/s)
850ms:    Bundle parsed (JS parsing: ~150ms)
950ms:    React.hydrateRoot() begins walking DOM
1100ms:   Hydration complete → page interactive
                ↓ HYDRATION GAP ENDS (880ms gap!)

During the 880ms gap:
  User input events are QUEUED (not lost, but not processed)
  When hydration completes: queued events fire
  "Click storms" from impatient users can cause unexpected behavior
```

### The Interaction Queue Problem

```javascript
// Events during hydration gap are queued by the browser
// After hydration: they all fire at once

// User clicks "Open Menu" 3 times during 880ms gap:
// - Click 1 (t=300ms): queued
// - Click 2 (t=500ms): queued
// - Click 3 (t=700ms): queued

// Hydration completes at t=1100ms:
// - All 3 click events fire
// - Menu opens, closes, opens again — unexpected!

// Solutions:
// 1. Drain input queue intentionally after hydration
// 2. Use event capture to detect clicks before hydration
// 3. Reduce hydration time (faster: less gap = fewer queued events)
```

---

## 3. Full Hydration — The Classic Approach

Full hydration rehydrates the entire component tree at once:

```javascript
// Server (renders entire HTML):
const html = renderToString(<App />); // entire tree → HTML

// Client (hydrates entire tree):
const root = document.getElementById("root");
hydrateRoot(root, <App />);
// React walks EVERY node in the tree
// Attaches event listeners to ALL interactive elements
// Reconciles virtual DOM with real DOM
// Cost: proportional to total number of components
```

### Full Hydration Timeline

```
100ms:  HTML received — FCP
        ↓
100ms:  Start downloading main bundle (300KB)
        ↓
400ms:  Bundle downloaded + parsed
        ↓
400ms:  hydrateRoot() begins
        Components: [A][B][C][D][E][F][G]...[N]
        React walks all N components in tree
        ↓
600ms:  Hydration complete — FULLY interactive
        Gap: 500ms (acceptable for most apps)

PROBLEM SCENARIO:
  Large app (500+ components, 1MB bundle):
  100ms: FCP
  1200ms: Bundle downloaded + parsed
  1200ms: hydrateRoot() begins
  1500ms: Hydration complete
  Gap: 1,400ms — users experience the page as "broken"
```

---

## 4. Progressive Hydration

Progressive hydration hydrates components in a priority order rather than all at once. High-priority, visible components hydrate first.

```jsx
// React 18: startTransition for lower-priority hydration
import { hydrateRoot, startTransition } from 'react-dom/client';

const root = hydrateRoot(document.getElementById('root'), <Shell />);

// High priority: visible, interactive content hydrates immediately
// Low priority: below-fold, non-interactive content hydrates during idle time

// Deferred hydration with startTransition
function DeferredHydration({ children }: { children: ReactNode }) {
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    startTransition(() => {
      setHydrated(true); // low-priority hydration
    });
  }, []);

  return hydrated ? children : <StaticFallback />;
}

// Usage:
function App() {
  return (
    <>
      {/* Hydrates immediately — above fold, interactive */}
      <Header />
      <HeroSection />

      {/* Hydrates after initial interaction is possible */}
      <DeferredHydration>
        <BelowFoldContent />
      </DeferredHydration>

      {/* Hydrates last — footer rarely needs JS */}
      <DeferredHydration>
        <Footer />
      </DeferredHydration>
    </>
  );
}
```

### Viewport-Based Progressive Hydration

```jsx
// Hydrate when element enters viewport
function LazyHydrate({ children, fallback }: LazyHydrateProps) {
  const ref = useRef<HTMLDivElement>(null);
  const [isHydrated, setHydrated] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setHydrated(true);
          observer.disconnect();
        }
      },
      { rootMargin: '200px' } // start hydrating 200px before visible
    );

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  if (isHydrated) {
    return <div ref={ref}>{children}</div>;
  }

  return (
    <div
      ref={ref}
      dangerouslySetInnerHTML={{ __html: '' }} // SSR content rendered by server
      suppressHydrationWarning
    />
  );
}

// Usage:
function ProductPage() {
  return (
    <>
      <ProductDetail />        {/* hydrates immediately */}
      <LazyHydrate fallback={<ReviewsSkeleton />}>
        <ReviewSection />      {/* hydrates when scrolled to */}
      </LazyHydrate>
      <LazyHydrate>
        <RelatedProducts />    {/* hydrates when scrolled to */}
      </LazyHydrate>
    </>
  );
}
```

---

## 5. Partial Hydration and Islands Architecture

Partial hydration hydrates ONLY the interactive components — static content never runs JavaScript on the client.

```
TRADITIONAL FULL HYDRATION:
  <Page>                          ← hydrated (JS) → useless (no interaction)
    <Header>                      ← hydrated (JS) → mostly static
      <Logo />                    ← hydrated (JS) → pure static HTML
      <Nav />                     ← hydrated (JS) → maybe 1-2 links
    </Header>
    <Article content={...} />     ← hydrated (JS) → 100% static text
    <CommentForm />               ← hydrated (JS) → ACTUALLY NEEDS JS
    <Footer />                    ← hydrated (JS) → pure static HTML
  </Page>

  JS for 90% static content: wasted

PARTIAL HYDRATION (Islands):
  <Page>                          ← NOT hydrated (no JS)
    <Header>                      ← NOT hydrated (no JS)
      <Logo />                    ← NOT hydrated (no JS)
      <Nav />                     ← NOT hydrated (no JS)
    </Header>
    <Article content={...} />     ← NOT hydrated (no JS)
    🏝 <CommentForm />            ← ISLAND: hydrated (this needs JS)
    <Footer />                    ← NOT hydrated (no JS)
  </Page>

  JS sent: only CommentForm (~15KB vs 300KB)
```

### Astro Islands Implementation

```astro
---
// src/pages/article/[slug].astro
const { slug } = Astro.params;
const article = await fetchArticle(slug);
---

<!-- Static HTML — zero JavaScript for these components -->
<header>
  <Logo />
  <Nav />
</header>

<main>
  <!-- Static: article content, no JS needed -->
  <article>
    <h1>{article.title}</h1>
    {article.content}
  </article>

  <!-- ISLAND: hydrate on load (interactive component) -->
  <CommentForm client:load articleId={article.id} />

  <!-- ISLAND: hydrate when visible (lazy) -->
  <RelatedArticles client:visible articleId={article.id} />

  <!-- ISLAND: hydrate on first user interaction -->
  <ShareMenu client:idle />
</main>

<footer>
  <!-- Static: no JS needed -->
  <FooterLinks />
</footer>
```

### DIY Islands in Next.js

```tsx
// Marking components as server-only vs client
// (Next.js App Router automatically does this with 'use client')

// Server Component (no JS shipped to client):
// app/article/page.tsx
export default async function ArticlePage({ params }) {
  const article = await fetchArticle(params.slug);

  return (
    <article>
      <h1>{article.title}</h1>
      <ArticleContent content={article.content} />
      {/* Client component: JavaScript IS shipped for this */}
      <CommentForm articleId={article.id} />
      {/* Client component */}
      <LikeButton articleId={article.id} />
    </article>
  );
}

// Client Component (JavaScript shipped, React hydration):
("use client");
// components/CommentForm.tsx
export function CommentForm({ articleId }) {
  const [comment, setComment] = useState("");
  // This component's JavaScript IS sent to the browser
  // and it DOES hydrate
}
```

---

## 6. Lazy / On-Demand Hydration

Hydrate components only when the user actually interacts with them:

```jsx
// Hydrate on first interaction with the element
function HydrateOnInteraction({
  children,
  events = ["click", "touchstart", "keydown"],
}) {
  const ref = useRef < HTMLDivElement > null;
  const [isHydrated, setHydrated] = useState(false);

  useEffect(() => {
    if (isHydrated) return;

    const element = ref.current;
    if (!element) return;

    function hydrate() {
      setHydrated(true);
      // Remove listeners after hydrating
      events.forEach((event) => element.removeEventListener(event, hydrate));
    }

    events.forEach((event) =>
      element.addEventListener(event, hydrate, { once: true, passive: true }),
    );
    return () =>
      events.forEach((event) => element.removeEventListener(event, hydrate));
  }, [isHydrated, events]);

  if (isHydrated) return <>{children}</>;

  // Render server-side HTML without hydration
  return (
    <div ref={ref} suppressHydrationWarning>
      {/* Server-rendered static HTML placeholder */}
    </div>
  );
}

// Usage: dropdown menu hydrates when user first hovers/clicks
function NavMenu() {
  return (
    <nav>
      <Link href="/">Home</Link>
      <HydrateOnInteraction events={["mouseenter", "focusin"]}>
        <DropdownMenu items={megaMenuItems} />
      </HydrateOnInteraction>
    </nav>
  );
}
```

### On-Demand Hydration with Preloading

```jsx
// Preload component code when near, hydrate on interaction
function SmartHydrate({ component: Component, preloadDistance = "200px" }) {
  const ref = useRef(null);
  const [preloaded, setPreloaded] = useState(false);
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    // PRELOAD: download JS when near viewport (using dynamic import)
    const preloader = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          import(/* @vite-ignore */ Component.importPath).then(() =>
            setPreloaded(true),
          );
          preloader.disconnect();
        }
      },
      { rootMargin: preloadDistance },
    );
    if (ref.current) preloader.observe(ref.current);

    // HYDRATE: activate when user interacts
    const activator = ref.current;
    activator?.addEventListener("click", () => setHydrated(true), {
      once: true,
    });

    return () => {
      preloader.disconnect();
    };
  }, []);

  if (hydrated) return <Component />;

  // Show static server-rendered HTML
  return (
    <div
      ref={ref}
      dangerouslySetInnerHTML={{ __html: "" }}
      suppressHydrationWarning
    />
  );
}
```

---

## 7. Streaming SSR and Selective Hydration

React 18's Streaming SSR allows parts of the page to be delivered and hydrated before the entire page is ready.

```jsx
// Server: stream HTML with Suspense boundaries
import { renderToPipeableStream } from "react-dom/server";

app.get("/product/:id", (req, res) => {
  const { pipe } = renderToPipeableStream(
    <App>
      <ProductLayout>
        {/* Streams immediately */}
        <ProductHeader productId={req.params.id} />

        {/* Streams when data is ready */}
        <Suspense fallback={<ReviewsSkeleton />}>
          <ProductReviews productId={req.params.id} />
        </Suspense>

        {/* Streams when data is ready */}
        <Suspense fallback={<RelatedSkeleton />}>
          <RelatedProducts productId={req.params.id} />
        </Suspense>
      </ProductLayout>
    </App>,
    {
      onShellReady() {
        // Shell (header, layout) ready — start streaming
        res.setHeader("Content-Type", "text/html");
        pipe(res);
      },
    },
  );
});
```

### Selective Hydration — Priority Based on Interaction

```jsx
// React 18: Suspense boundaries enable selective hydration
// React prioritizes hydrating components the user is interacting with

function App() {
  return (
    <>
      {/* No Suspense: hydrates as part of initial bundle */}
      <Header />

      {/* Suspense: independently streamable AND hydration-prioritizable */}
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>

      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
    </>
  );
}

// If user clicks Sidebar before it's hydrated:
// React immediately prioritizes hydrating Sidebar over MainContent
// User's click is processed as soon as Sidebar is hydrated
// MainContent hydration is deferred
```

---

## 8. Resumability — Hydration Without Replay

Resumability (Qwik's approach) eliminates hydration by serializing component state into the HTML. Instead of replaying component logic, the client "resumes" from the server's serialized state.

```
HYDRATION (React, Vue, etc.):
  Server: execute components → HTML
  Client: download ALL component code → execute AGAIN → match DOM → attach listeners

  Problem: duplicate execution (server + client)
  Problem: must download code for ALL components, even static ones

RESUMABILITY (Qwik):
  Server: execute components → HTML + serialize state INTO the HTML
  Client: download ONLY the code needed for the current interaction

  HTML encodes: "if user clicks this button, here's the lazy-loaded handler"
  No replay — pick up exactly where server left off
```

```html
<!-- Qwik serialized HTML (simplified) -->
<button on:click="/chunks/button-handler-abc.js#handleClick" q:obj="1">
  Click me
</button>
<!--
  When user clicks:
  1. Browser downloads button-handler-abc.js (~2KB)
  2. Executes only handleClick
  3. No full framework initialization needed
-->
```

### Resumability vs Hydration

```
HYDRATION (React):
  TTI cost: O(components in tree) × parsing/execution cost
  Benefit: familiar React model, mature ecosystem
  Cost: must download and execute all component code upfront

RESUMABILITY (Qwik):
  TTI cost: O(1) — near-zero startup (no component re-execution)
  Benefit: pages interactive almost immediately
  Cost: unfamiliar model, smaller ecosystem, serialization overhead in HTML

When resumability shines:
  ✓ Content-heavy pages with few interactive components
  ✓ E-commerce product pages
  ✓ Blog/news articles

When hydration is fine:
  ✓ Highly interactive apps (dashboards, editors)
  ✓ Small component trees
  ✓ Teams deeply invested in React
```

---

## 9. Hydration Mismatches

A hydration mismatch occurs when the HTML the server rendered differs from what React generates on the client:

```
Server renders: <p>Hello, Alice!</p>
Client renders: <p>Hello, Bob!</p>  (different user session)

React: MISMATCH! Must either:
  a) Throw away server HTML and re-render from scratch (bad: loses SSR benefit)
  b) Patch the specific difference (only in React 18 with suppressHydrationWarning)
  c) Crash with error (React strict mode)
```

### Common Causes of Mismatches

```jsx
// 1. DATES AND TIMES
function TimeStamp() {
  return <time>{new Date().toLocaleString()}</time>;
  // Server: "Jan 15, 2024, 10:30:00 AM"
  // Client (100ms later): "Jan 15, 2024, 10:30:01 AM"
  // → MISMATCH
}

// Fix: use useEffect to update after mount
function TimeStamp() {
  const [time, setTime] = (useState < string) | (null > null);
  useEffect(() => {
    setTime(new Date().toLocaleString());
  }, []);
  return <time>{time ?? "..."}</time>;
}

// 2. RANDOM VALUES
function RandomId() {
  return <div id={Math.random().toString(36)}>Content</div>;
  // Server: id="0.abc123"
  // Client: id="0.xyz789"
  // → MISMATCH
}

// Fix: stable ID generation
const id = useId(); // React 18: generates stable, consistent IDs

// 3. BROWSER-ONLY APIS
function WindowSize() {
  return <p>Window: {window.innerWidth}px</p>;
  // Server: window is undefined → throws
}

// Fix: guard with mounted check
function WindowSize() {
  const [width, setWidth] = (useState < number) | (null > null);
  useEffect(() => {
    setWidth(window.innerWidth);
  }, []);
  return <p>Window: {width !== null ? `${width}px` : "..."}</p>;
}

// 4. USER-SPECIFIC CONTENT
function UserGreeting() {
  const user = getCookieUser(); // reads client cookie
  return <p>Hello, {user.name}!</p>;
  // Server: no cookie context → user is null → "Hello, !"
  // Client: cookie exists → "Hello, Alice!"
  // → MISMATCH
}

// Fix: don't render user-specific content during SSR
function UserGreeting() {
  const [user, setUser] = (useState < User) | (null > null);
  useEffect(() => {
    setUser(getClientUser());
  }, []);
  if (!user) return <p>Hello!</p>; // SSR-safe fallback
  return <p>Hello, {user.name}!</p>;
}
```

### Suppressing Mismatch Warnings

```jsx
// suppressHydrationWarning: opt-out for intentionally different server/client content
function LastUpdated({ timestamp }) {
  return (
    <time
      dateTime={timestamp}
      suppressHydrationWarning // ← tells React: mismatch expected here, don't warn
    >
      {formatRelativeTime(timestamp)} {/* "2 hours ago" — differs by time */}
    </time>
  );
}
```

---

## 10. React Server Components — Beyond Hydration

React Server Components (RSC) eliminate hydration for server-only components entirely:

```
SERVER COMPONENT:
  Runs on server only
  Code NEVER shipped to client
  Cannot use: useState, useEffect, browser APIs, event handlers
  Can use: async/await, database queries, server-only libraries

CLIENT COMPONENT:
  JavaScript shipped to browser
  Hydrated in the traditional sense
  Can use: all hooks, event handlers, browser APIs

THE KEY INSIGHT:
  Traditional SSR: server renders → client re-executes to hydrate
  RSC: server renders → client receives rendered output (no code to re-execute)

  Server Component hydration cost: $0 (no client code, no hydration)
  Client Component hydration cost: normal (only for actual interactive components)
```

```tsx
// app/product/page.tsx (Server Component)
export default async function ProductPage({ params }) {
  // Runs ONLY on server — never on client
  const product = await db.products.findById(params.id);
  const reviews = await db.reviews.findByProduct(params.id);

  return (
    <div>
      {/* Server-rendered static output — no client JS */}
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <ProductImages images={product.images} />

      {/* Client Component: JavaScript shipped, hydrated */}
      <AddToCartButton productId={product.id} price={product.price} />

      {/* Server Component: renders to HTML, no client JS */}
      {reviews.map((review) => (
        <ReviewCard key={review.id} review={review} />
      ))}
    </div>
  );
}

// The entire page: ProductPage, ProductImages, ReviewCard
//   → pure HTML, zero client JS for these
// Only AddToCartButton → ~5KB of JavaScript shipped
// Hydration: only AddToCartButton needs hydration
```

---

## 11. Measuring Hydration Performance

### Timing Hydration

```javascript
// Measure hydration time
const hydrationStart = performance.now();

hydrateRoot(document.getElementById("root"), <App />, {
  onRecoverableError(err) {
    console.error("Hydration error:", err);
  },
});

// React 18: use scheduler to detect when hydration completes
import { startTransition } from "react";

startTransition(() => {
  // This runs after initial hydration
  const hydrationTime = performance.now() - hydrationStart;
  console.log(`Initial hydration: ${hydrationTime.toFixed(0)}ms`);

  analytics.track("hydration_time", { ms: hydrationTime });
});
```

### Using Web Vitals for Hydration Impact

```javascript
import { onFID, onINP, onTTFB, onFCP } from "web-vitals";

// FCP: when server-rendered content first appears
onFCP(({ value }) => {
  console.log(`FCP: ${value}ms`); // goal: < 1800ms
});

// INP: interaction responsiveness — directly impacted by hydration
onINP(({ value, rating, attribution }) => {
  console.log(`INP: ${value}ms (${rating})`);
  // During hydration: INP will be poor → input queued
  // After hydration: INP improves
});
```

### React DevTools Profiler for Hydration

```
React DevTools → Profiler tab → Record → Reload page → Stop

In the flame chart:
  "hydrateRoot" entry → total hydration time
  Individual component entries → which components took longest
  "Committed at: Xms" → when hydration completed

Look for:
  Long individual components (> 5ms) → candidates for lazy hydration
  Total hydration time > 500ms → need hydration optimization
```

---

## 12. Good Practices

### ✅ Hydrate interactivity lazily where possible

```jsx
// ✅ Only hydrate when user is near or interacting with the component
<LazyHydrate onInteraction={["click", "focusin"]}>
  <ComplexModal />
</LazyHydrate>
// Result: ComplexModal's 50KB of JS deferred until needed
```

### ✅ Keep server and client state consistent to avoid mismatches

```jsx
// ✅ Defensive pattern: always provide SSR-safe initial state
function Component() {
  // ❌ const value = window.localStorage.getItem('key'); // breaks SSR
  // ✅ Read browser APIs only after mount
  const [value, setValue] = (useState < string) | (null > null);
  useEffect(() => {
    setValue(window.localStorage.getItem("key"));
  }, []);
}
```

### ✅ Use `useId` for stable IDs across server/client

```jsx
// ✅ React 18: useId generates stable, consistent IDs
function FormField({ label }) {
  const id = useId(); // same value on server AND client
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

### ✅ Use Suspense boundaries for progressive hydration control

```jsx
// ✅ Suspense boundaries = hydration units
// React 18 can prioritize and selectively hydrate
<Suspense fallback={<Skeleton />}>
  <ImportantSection /> {/* hydrates first if interacted with */}
</Suspense>
```

---

## 13. Bad Practices

### ❌ Rendering user-specific data without mismatch guards

```jsx
// ❌ Causes hydration mismatch on every page load
function Greeting() {
  const user = useAuthStore((s) => s.user); // may not be available during SSR
  return <p>Hello, {user?.name ?? "Guest"}!</p>;
  // Server: "Hello, Guest!"
  // Client: "Hello, Alice!" after auth state loads → MISMATCH
}
```

### ❌ Hydrating the entire page when only 10% needs JavaScript

```javascript
// ❌ Full hydration for a mostly-static page
hydrateRoot(document.getElementById("root"), <EntireApp />);
// Pays hydration cost for the navigation, header, article,
// footer — all of which are 100% static
```

### ❌ Suppressing hydration warnings without understanding why they exist

```jsx
// ❌ blindly suppressing without fixing the root cause
<div suppressHydrationWarning>
  {/* Hidden mismatch that causes real bugs */}
  {new Date().toISOString()}
</div>
```

---

## 14. Common Mistakes

### Mistake 1 — Not handling the "Zombie" hydration state

```jsx
// The user sees the page, clicks a button, nothing happens.
// They're confused because the page looked interactive.
// This "zombie" state is the hydration gap.

// ❌ No indication that hydration is pending:
function App() {
  return (
    <>
      <Hero />
      <InteractiveMap /> {/* looks interactive but isn't for 800ms */}
    </>
  );
}

// ✅ Indicate loading state during hydration
function InteractiveMap() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);

  if (!mounted) {
    return <StaticMapFallback />;
    // or return <MapSkeleton />;
    // Makes clear this is still loading
  }
  return <FullInteractiveMap />;
}
```

### Mistake 2 — Code-splitting without accounting for hydration

```javascript
// ❌ Dynamic import in React causes hydration mismatches if not handled
const HeavyChart = React.lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <HeavyChart />{" "}
      {/* Server: renders ChartSkeleton, Client: renders HeavyChart */}
    </Suspense>
  );
}
// Server: renders <ChartSkeleton /> because lazy isn't resolved server-side
// Client: hydrates expecting ChartSkeleton then shows HeavyChart
// Works in React 18 with streaming, but requires correct Suspense setup
```

### Mistake 3 — Excessive use of `suppressHydrationWarning`

```jsx
// ❌ Suppressing everything to hide problems
function Component() {
  return (
    <div suppressHydrationWarning>
      {/* Many elements with suppressHydrationWarning */}
      <span suppressHydrationWarning>{getClientData()}</span>
      <p suppressHydrationWarning>{new Date().toString()}</p>
    </div>
  );
}
// suppressHydrationWarning on parent suppresses ALL mismatch warnings
// Real bugs become hidden

// ✅ Fix the actual cause of the mismatch instead
```

---

## 15. Interview-Level Explanation

> **"What is hydration? What are the performance implications and how do you address them?"**

**Strong answer:**

> "Hydration is the process of attaching JavaScript interactivity to HTML that was rendered on the server. When SSR delivers HTML to the browser, the user sees content immediately — but clicking or typing does nothing because there are no event listeners. Hydration is React walking the entire DOM tree, matching it to a virtual DOM generated by re-running all the component code client-side, and attaching event handlers. Only after hydration completes is the page actually interactive.
>
> The performance problem is the hydration gap — the time between First Contentful Paint (when the SSR HTML appears) and Time to Interactive (when hydration completes). For a typical React app with a 500KB bundle, this gap can be 500-1500ms. During that time the page looks interactive but isn't, which users experience as the page being broken. Input events are queued and can fire in unexpected batches after hydration.
>
> The solutions span a spectrum of complexity. At the simple end, code-splitting reduces the JS bundle, which shrinks the hydration gap. Streaming SSR with React 18 lets parts of the page hydrate as they arrive rather than waiting for the complete page. Selective hydration prioritizes hydrating whatever the user is currently interacting with.
>
> Islands architecture is more aggressive: identify exactly which components need JavaScript, hydrate only those, and let the rest remain pure HTML. For a blog post page, that might mean only hydrating the comment form (15KB) rather than the entire app (300KB). Progressive hydration goes further: defer hydrating below-fold components until the user scrolls to them.
>
> Resumability, as implemented by Qwik, takes a fundamentally different approach: the server serializes all component state into the HTML, and the client 'resumes' from that state rather than replaying component execution. The client downloads only the code for interactions the user actually performs. For content-heavy pages, this can achieve near-zero JS startup cost.
>
> React Server Components represent the current direction for React: components that run only on the server never ship their JavaScript to the client at all, eliminating hydration cost for those components entirely. Only the truly interactive pieces — forms, buttons, carousels — need client-side JavaScript and hydration."

---

## 16. Exercises

### Exercise 1 — Fix the hydration mismatches

```jsx
// Find and fix all hydration mismatches in this component:
function UserDashboard() {
  const theme = localStorage.getItem("theme") ?? "light";
  const sessionId = Math.random().toString(36).slice(2);
  const now = new Date().toLocaleTimeString();
  const userId = document.cookie.match(/userId=(\w+)/)?.[1];

  return (
    <div className={`dashboard dashboard--${theme}`} data-session={sessionId}>
      <p>Current time: {now}</p>
      <p>User: {userId ?? "anonymous"}</p>
    </div>
  );
}
```

<details>
<summary>Solution</summary>

```jsx
function UserDashboard() {
  // Fix 1: localStorage - not available during SSR
  const [theme, setTheme] = useState < string > "light";

  // Fix 2: random values - different server/client
  const sessionId = useId(); // stable across server/client

  // Fix 3: time - different server/client
  const [now, setNow] = (useState < string) | (null > null);

  // Fix 4: document.cookie - not available during SSR
  const [userId, setUserId] = (useState < string) | (null > null);

  useEffect(() => {
    // All browser-only reads happen after mount
    setTheme(localStorage.getItem("theme") ?? "light");
    setNow(new Date().toLocaleTimeString());
    setUserId(document.cookie.match(/userId=(\w+)/)?.[1] ?? null);

    // Update time every second
    const interval = setInterval(() => {
      setNow(new Date().toLocaleTimeString());
    }, 1000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div className={`dashboard dashboard--${theme}`} data-session={sessionId}>
      <p>Current time: {now ?? "..."}</p>
      <p>User: {userId ?? "anonymous"}</p>
    </div>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/10-ssr-csr-isr-streaming.md`](../browser-internals/10-ssr-csr-isr-streaming.md) — Rendering strategies overview
- [`rendering/02-virtual-dom.md`](./02-virtual-dom.md) — Virtual DOM that hydration uses
- [`rendering/03-cooperative-scheduling.md`](./03-cooperative-scheduling.md) — Fiber scheduling during hydration
- [`system-design/02-feature-based-structure.md`](../system-design/02-feature-based-structure.md) — Islands architecture at module level

---

<div align="center">

**`rendering/` section complete!** 🎉

All 5 rendering files done:
`01-dom-batching.md` · `02-virtual-dom.md` · `03-cooperative-scheduling.md` · `04-paint-optimization.md` · **`05-hydration-patterns.md`** ✓

**Next section:** [`caching/`](../caching/) →

</div>
