# 08 — Critical Rendering Path

> **"The Critical Rendering Path is the minimum work the browser must complete before showing a single pixel to the user. Every millisecond you shave off it is a millisecond less of blank screen. Optimizing it is the highest-leverage performance work you can do."**

The Critical Rendering Path (CRP) describes the sequence of steps the browser must complete to render the first visible content. It's the direct path from raw HTML bytes to the first painted pixels. Optimizing the CRP is the foundation of all initial load performance work — it directly drives First Contentful Paint (FCP), Largest Contentful Paint (LCP), and Interaction to Next Paint (INP). This document covers every step, what makes each one slow, and systematic techniques to minimize time to first render.

---

## 📚 Table of Contents

1. [What the Critical Rendering Path Is](#1-what-the-critical-rendering-path-is)
2. [The Six Steps of the CRP](#2-the-six-steps-of-the-crp)
3. [Render-Blocking Resources](#3-render-blocking-resources)
4. [Parser-Blocking Scripts](#4-parser-blocking-scripts)
5. [Measuring the CRP](#5-measuring-the-crp)
6. [CRP Optimization Strategies](#6-crp-optimization-strategies)
7. [Critical CSS — The Most Impactful Optimization](#7-critical-css--the-most-impactful-optimization)
8. [Resource Prioritization](#8-resource-prioritization)
9. [HTTP/2 and CRP](#9-http2-and-crp)
10. [The Network Waterfall](#10-the-network-waterfall)
11. [Core Web Vitals Connection](#11-core-web-vitals-connection)
12. [Server-Side Optimizations](#12-server-side-optimizations)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. What the Critical Rendering Path Is

The CRP is the chain of steps — from server to screen — that the browser must complete before displaying any visual content. It's "critical" because every step is necessary and nothing can be skipped.

```
User types URL                       t = 0ms
       │
       ▼
DNS resolution                       t = 0–100ms
       │
       ▼
TCP + TLS handshake                  t = 0–200ms (1–2 RTTs)
       │
       ▼
HTTP request sent                    t = varies
       │
       ▼
Server processes + responds          t = TTFB (Time to First Byte)
       │
       ▼
HTML bytes arrive (streaming)        t = TTFB + transfer time
       │
       ▼
HTML parsing → DOM construction      t = ongoing as bytes arrive
       │
       ├──────────────────────────────►  CSS download + parse → CSSOM
       │                               (blocks render)
       │
       ▼  (both DOM and CSSOM ready)
Render tree construction
       │
       ▼
Layout (geometry computation)
       │
       ▼
Paint (rasterization)
       │
       ▼
Composite → First pixel on screen ← FIRST CONTENTFUL PAINT (FCP)
```

**Everything from DNS to First Paint is the Critical Rendering Path.**

The goal: minimize the time between `t = 0` (user intent) and First Paint.

---

## 2. The Six Steps of the CRP

### Step 1 — Build the DOM

```
Input:  HTML bytes from network
Output: DOM tree
Bottlenecks:
  - Network speed (HTML download time)
  - HTML file size (large HTML takes longer to parse)
  - Parser-blocking scripts (stop DOM construction)

Optimization levers:
  - Reduce HTML size (compression, server-side rendering efficiency)
  - Defer or async scripts
  - Remove render-blocking scripts from <head>
```

### Step 2 — Build the CSSOM

```
Input:  CSS bytes from network
Output: CSSOM tree
Bottlenecks:
  - Network speed (CSS download time)
  - CSS file size
  - Number of CSS files (each is a separate request)
  - CSS parsing time (complex selectors, large files)

Optimization levers:
  - Reduce CSS size (minification, unused CSS removal)
  - Inline critical CSS (eliminate network request)
  - Load non-critical CSS asynchronously
  - Use media queries to avoid unnecessary blocking
```

### Step 3 — Build the Render Tree

```
Input:  DOM + CSSOM
Output: Render tree (visible elements with computed styles)
Bottlenecks:
  - Both DOM and CSSOM must be complete
  - Any CSSOM delay directly delays render tree

Note: Render tree construction itself is fast — milliseconds.
The bottleneck is WAITING for both DOM and CSSOM.
```

### Step 4 — Layout

```
Input:  Render tree
Output: Box model geometry for all elements
Bottlenecks:
  - Complex layouts (deep nesting, flexible box, grid)
  - Large number of elements
  - Percentage/relative sizing (requires parent computation)

Typically: 2–10ms for initial layout of a typical page
```

### Step 5 — Paint

```
Input:  Layout tree
Output: Layer bitmaps
Bottlenecks:
  - Complex visual effects (shadows, gradients, blur)
  - Large page area
  - Many compositor layers

Typically: 2–20ms for initial paint
```

### Step 6 — Composite

```
Input:  Layer bitmaps
Output: Final screen frame
Bottlenecks:
  - Number and size of compositor layers
  - GPU memory pressure

Typically: 1–5ms
```

---

## 3. Render-Blocking Resources

A **render-blocking resource** prevents the browser from proceeding to the render tree construction step until it's fully downloaded and processed.

### What Blocks Rendering

```html
<!-- ✅ Render-blocking (expected and necessary) -->
<link rel="stylesheet" href="styles.css" />

<!-- ✅ Not render-blocking — wrong media type for current context -->
<link rel="stylesheet" href="print.css" media="print" />

<!-- ❌ Render-blocking script in <head> -->
<script src="app.js"></script>

<!-- ✅ Not render-blocking — async -->
<script async src="analytics.js"></script>

<!-- ✅ Not render-blocking — defer (but runs after DOM parse) -->
<script defer src="app.js"></script>
```

### The Blocking Sequence

```
HTML parser reaches: <link rel="stylesheet" href="styles.css">

Timeline:
  CSS request:  ─────────────────────── (downloading)
  HTML parsing: ──────────────────────────────────────── (continues)
  Render:                                ↑ BLOCKED until CSS ready
                                         (can't build render tree)

t=0ms: CSS request dispatched
t=0ms: HTML parser continues (parsing NOT blocked by CSS)
t=200ms: CSS downloaded and parsed → CSSOM complete
t=200ms: Render tree can now be built
         (if DOM is ready — which it should be for typical pages)
t=202ms: Layout
t=205ms: Paint → FIRST PIXEL
```

### Why CSS Can't Be Made Non-Blocking (for render)

CSS is render-blocking by design because of the cascade:

```css
/* CSS rule at line 50,000: */
* {
  visibility: hidden;
}
/* This would hide EVERYTHING if browser rendered before seeing this rule */
/* So browser MUST see all CSS before rendering anything */
```

The fix is not to make CSS non-blocking — it's to make CSS smaller and deliver it faster.

---

## 4. Parser-Blocking Scripts

Scripts are the more severe type of blocking — they stop both parsing AND rendering.

### Why Scripts Block the Parser

```html
<script src="data-injector.js"></script>
<!-- parser stops here until script downloads and executes -->
<!-- Why: script might call document.write(), adding more HTML to parse -->

<script>
  document.write('<link rel="stylesheet" href="extra.css">');
  // This inserts new HTML into the parse stream
  // Parser must let script run first to discover this
</script>
```

### The Blocking Timeline

```
HTML parsing: ████████░░░░░░░░░░░░░░░░░░░░░████████████████
                       ↑                  ↑
              parser stops         script finishes
              (script tag found)   parser resumes

Script download:        ░░░░░░░░░░░░
Script execute:                     ░░
CSS/images below: not requested until parser sees them

PROBLEM: script delays discovery of all resources below it
```

### Script Loading Comparison

```
<script src="app.js">:
  Download:  blocking (parser stops)
  Execute:   blocking (in-order)
  DOM ready: NO (must wait for script to execute)

<script async src="app.js">:
  Download:  parallel (parser continues)
  Execute:   whenever downloaded (may interrupt parsing)
  DOM ready: NO (if executes before parsing completes)
  Use for:   independent scripts (analytics, ads)

<script defer src="app.js">:
  Download:  parallel (parser continues)
  Execute:   after DOM fully parsed, in document order
  DOM ready: YES (DOM is complete when deferred scripts run)
  Use for:   most application scripts

<script type="module">:
  Download:  parallel (like defer)
  Execute:   after DOM fully parsed (like defer)
  DOM ready: YES
  Use for:   ES module scripts
```

---

## 5. Measuring the CRP

### Navigation Timing API

```javascript
// The most comprehensive source of CRP metrics
const [navEntry] = performance.getEntriesByType("navigation");

const metrics = {
  // Network timing
  dnsLookup: navEntry.domainLookupEnd - navEntry.domainLookupStart,
  tcpConnect: navEntry.connectEnd - navEntry.connectStart,
  tlsHandshake: navEntry.requestStart - navEntry.secureConnectionStart,
  ttfb: navEntry.responseStart - navEntry.requestStart,
  download: navEntry.responseEnd - navEntry.responseStart,

  // DOM timing
  domParsing: navEntry.domInteractive - navEntry.responseEnd,
  deferredScripts:
    navEntry.domContentLoadedEventStart - navEntry.domInteractive,
  domComplete: navEntry.domComplete - navEntry.responseEnd,

  // Key milestones
  domInteractive: navEntry.domInteractive, // HTML parsed
  dcl: navEntry.domContentLoadedEventEnd, // DCL fired
  load: navEntry.loadEventEnd, // load fired
};

console.table(metrics);
```

### Paint Timing API

```javascript
// First Contentful Paint (FCP) — first CRP completion
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.name, entry.startTime.toFixed(0) + "ms");
    // "first-paint": 380ms
    // "first-contentful-paint": 420ms
  }
});
observer.observe({ entryTypes: ["paint"] });
```

### Largest Contentful Paint

```javascript
const lcpObserver = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lcp = entries[entries.length - 1]; // latest LCP candidate
  console.log("LCP:", lcp.startTime.toFixed(0) + "ms");
  console.log("LCP element:", lcp.element);
  console.log("LCP size:", lcp.size);
});
lcpObserver.observe({ entryTypes: ["largest-contentful-paint"] });
```

### CRP Visualization in DevTools

```
DevTools → Network panel → set throttling (e.g., Slow 3G)
  → Reload page
  → Timeline shows waterfall with:
      Blue bar:   HTML download
      Purple bar: CSS download
      Yellow bar: Script download
      Green bar:  DOM parsing
      Orange bar: Layout/paint

Look for:
  - "Blocking time" (hatched pattern): resource waiting in queue
  - Long chains: resource B cannot start until resource A finishes
  - Script/CSS blocking the waterfall
```

---

## 6. CRP Optimization Strategies

### Strategy 1 — Reduce Critical Resources

```
Critical resource = must be downloaded before first render

Before optimization:
  Critical path: HTML → styles.css (200KB) → main.js (500KB) → render
  Critical path length: 3 resources, 900KB, 3 RTTs minimum

After optimization:
  Critical path: HTML + inlined critical CSS → render
  Critical path length: 1 resource, 50KB, 1 RTT minimum
```

### Strategy 2 — Reduce Critical Bytes

```
Every byte on the critical path delays render.
Minimum critical bytes = HTML + critical CSS.

Reduce with:
  - Minification (remove whitespace, comments)
  - Compression (gzip, Brotli) — 60-80% reduction
  - Unused CSS removal (PurgeCSS, UnCSS)
  - Split CSS: inline critical, async non-critical
```

### Strategy 3 — Reduce Critical Path Length (RTTs)

```
Round Trip Time (RTT): time for a request to travel to server and back

Waterfall without optimization (each resource requires a new RTT):
  t=0:    HTML request
  t=100:  HTML received → discover CSS reference
  t=100:  CSS request
  t=200:  CSS received → discover JS reference
  t=200:  JS request
  t=300:  JS received → render begins

  Total: 3 RTTs = 300ms minimum (ignoring download time)

With <link rel="preload">:
  t=0:    HTML request
  t=100:  HTML received → parse preload hints → request CSS AND JS simultaneously
  t=200:  CSS AND JS received → render begins

  Total: 2 RTTs = 200ms minimum (parallelized)
```

### Strategy 4 — Eliminate Render Blockers

```
Before:
  <head>
    <link rel="stylesheet" href="all-styles.css">  ← render-blocking
    <script src="analytics.js"></script>            ← parser-blocking
    <script src="app.js"></script>                  ← parser-blocking
  </head>

  Render doesn't begin until: all-styles.css + analytics.js + app.js downloaded

After:
  <head>
    <style>[critical CSS inlined]</style>  ← no network request
    <link rel="preload" href="all-styles.css" as="style" onload="this.rel='stylesheet'">
    <script async src="analytics.js"></script>   ← no blocking
    <script defer src="app.js"></script>          ← no blocking
  </head>

  Render begins immediately after: HTML + inlined critical CSS parsed
```

---

## 7. Critical CSS — The Most Impactful Optimization

Inlining "critical CSS" (the styles needed for above-the-fold content) eliminates the CSS network request from the critical path.

### What Is Critical CSS

```
Critical CSS = the minimum CSS needed to render
               what the user sees on first load
               (above-the-fold content)

Includes:
  - Body/html base styles
  - Header/navigation styles
  - Hero/above-fold component styles
  - Font declarations for visible text

Does NOT include:
  - Styles for below-fold content
  - Hover/focus states (user hasn't interacted yet)
  - Modal/overlay styles (not visible on load)
  - Animation keyframes (for animations that start later)
```

### Critical CSS in Action

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Page</title>

    <!-- ✅ Critical CSS inlined — no network request -->
    <style>
      /* Reset */
      *,
      *::before,
      *::after {
        box-sizing: border-box;
        margin: 0;
      }

      /* Typography basics */
      body {
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
        line-height: 1.5;
        color: #1a1a1a;
      }

      /* Navigation */
      .nav {
        display: flex;
        align-items: center;
        padding: 1rem 2rem;
        background: #fff;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
      }
      .nav__logo {
        font-weight: 700;
        font-size: 1.25rem;
      }

      /* Hero section */
      .hero {
        padding: 6rem 2rem;
        background: linear-gradient(135deg, #667eea, #764ba2);
        color: #fff;
        text-align: center;
      }
      .hero__title {
        font-size: 3rem;
        font-weight: 700;
        margin-bottom: 1rem;
      }
      .hero__subtitle {
        font-size: 1.25rem;
        opacity: 0.9;
      }
      .hero__cta {
        display: inline-block;
        margin-top: 2rem;
        padding: 1rem 2rem;
        background: #fff;
        color: #764ba2;
        border-radius: 4px;
        text-decoration: none;
        font-weight: 600;
      }
    </style>

    <!-- Full CSS — async, doesn't block render -->
    <link
      rel="preload"
      href="/styles/full.css"
      as="style"
      onload="this.onload=null;this.rel='stylesheet'"
    />
    <noscript><link rel="stylesheet" href="/styles/full.css" /></noscript>
  </head>
  <body>
    <nav class="nav">
      <span class="nav__logo">Brand</span>
      <!-- nav items... -->
    </nav>

    <section class="hero">
      <h1 class="hero__title">Welcome</h1>
      <p class="hero__subtitle">Build amazing things.</p>
      <a href="/signup" class="hero__cta">Get Started</a>
    </section>

    <!-- Below-fold content renders after full.css loads -->
    <!-- User doesn't notice — it's below the fold -->
  </body>
</html>
```

### Automated Critical CSS Extraction

```javascript
// Tools for automated critical CSS extraction:

// 1. Critical (npm package by Addy Osmani)
const critical = require("critical");
await critical.generate({
  src: "index.html",
  target: {
    html: "dist/index.html",
    css: "dist/critical.css",
    uncritical: "dist/async.css",
  },
  width: 1300,
  height: 900,
  inline: true,
});

// 2. Penthouse (used by critical internally)
const penthouse = require("penthouse");
const criticalCSS = await penthouse({
  url: "http://localhost:3000",
  css: "./styles/main.css",
  width: 1300,
  height: 900,
});
```

---

## 8. Resource Prioritization

Browsers fetch resources with different priorities. Understanding and controlling priorities is key to CRP optimization.

### Default Priority Ordering

```
Priority  Resource Type
────────────────────────────────────────────────────────
Highest:  Main document (HTML)
          Stylesheets (in <head>)
          Scripts in <head> (no async/defer)
          Preloaded fonts

High:     Scripts in <body>
          Images in viewport (first LCP candidate)
          Preloaded resources

Medium:   Images below fold
          Async scripts

Low:      Deferred scripts
          Images in hidden elements

Lowest:   Prefetched resources
          Background images
```

### `<link rel="preload">` — Bump a Resource's Priority

```html
<!-- Tell the browser about critical resources it would discover late -->

<!-- Preload hero image — discovered in CSS, not HTML: browser doesn't know it's critical -->
<link rel="preload" href="/images/hero.webp" as="image" />

<!-- Preload font — critical for FCP text rendering -->
<link
  rel="preload"
  href="/fonts/inter-400.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>

<!-- Preload LCP image -->
<link
  rel="preload"
  href="/images/product-hero.jpg"
  as="image"
  fetchpriority="high"
/>

<!-- Preload critical JS -->
<link rel="preload" href="/js/critical-bundle.js" as="script" />
```

### `fetchpriority` — Explicit Priority Hints

```html
<!-- High priority: LCP image -->
<img src="hero.jpg" fetchpriority="high" alt="Hero" />

<!-- Low priority: below-fold images -->
<img src="product-1.jpg" fetchpriority="low" loading="lazy" alt="Product" />

<!-- High priority: critical script -->
<script src="critical.js" fetchpriority="high"></script>

<!-- Low priority: non-critical -->
<script src="analytics.js" fetchpriority="low" async></script>
```

### `<link rel="preconnect">` — Early Connection

```html
<!-- Establish connection early to external origins -->
<!-- Saves 100-300ms per external origin on first resource request -->

<!-- For Google Fonts: -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- For CDN: -->
<link rel="preconnect" href="https://cdn.example.com" />

<!-- For API server: -->
<link rel="preconnect" href="https://api.example.com" />
```

### `<link rel="dns-prefetch">` — Lighter Weight Alternative

```html
<!-- DNS resolution only (not full connection) -->
<!-- Cheaper than preconnect, less network overhead -->
<!-- Use when you need the origin but don't yet know the specific resource -->
<link rel="dns-prefetch" href="https://analytics.example.com" />
<link rel="dns-prefetch" href="https://fonts.gstatic.com" />
```

### `<link rel="prefetch">` — Future Navigation Resources

```html
<!-- Resources for the NEXT page (not current CRP) -->
<!-- Low priority, loads in idle time -->
<link rel="prefetch" href="/dashboard.js" />
<link rel="prefetch" href="/dashboard.css" />
<!-- When user navigates to /dashboard: resources already in browser cache -->
```

---

## 9. HTTP/2 and CRP

HTTP/2 changes CRP optimization strategies significantly. Understanding the differences matters for choosing the right techniques.

### HTTP/1.1 Problems

```
HTTP/1.1 limitations affecting CRP:

1. Head-of-line blocking per connection:
   Only one request at a time per connection

2. Limited connections:
   Browser opens 6 connections max per origin
   6 resources can download in parallel
   7th resource must wait for one of the 6 to finish

3. Request overhead:
   Each request repeats large HTTP headers
   Performance cost for many small files

Workarounds (outdated with HTTP/2):
  - CSS/JS bundling (reduce request count)
  - Domain sharding (spread resources across subdomains)
  - Image sprites (combine images into one)
```

### HTTP/2 Advantages

```
HTTP/2 solutions:

1. Multiplexing:
   Multiple requests on ONE connection, simultaneously
   No head-of-line blocking at HTTP layer
   Many small files as efficient as one large file

2. Header compression (HPACK):
   Repeated headers compressed
   Reduces overhead for many requests

3. Server Push (deprecated but existed):
   Server sends resources before client asks
   (largely replaced by preload hints)

4. Stream prioritization:
   Browser tells server resource priority
   Critical CSS before non-critical scripts
```

### CRP Implications of HTTP/2

```
HTTP/1.1 CRP strategy: BUNDLE EVERYTHING
  - One large CSS file (minimize requests)
  - One large JS bundle (minimize requests)
  - Sprites, data URIs (eliminate requests)
  Rationale: each request costs an RTT

HTTP/2 CRP strategy: SPLIT STRATEGICALLY
  - Split CSS: critical (inlined) + non-critical (async)
  - Split JS: critical-path (preloaded) + lazy (imported)
  - Use route-based code splitting (each route has its own bundle)
  - Many small files = same performance, better caching granularity
  Rationale: multiplexing makes requests cheap
```

---

## 10. The Network Waterfall

Understanding the network waterfall is essential for CRP analysis.

### Reading the Waterfall

```
DevTools Network panel (waterfall view):

Resource          Size    Waterfall
──────────────────────────────────────────────────────────────
index.html        12KB    ├══════════════════════╗ ← blocking
styles.css        45KB    │              ├═══════╗ ← blocked by HTML parse
app.js            120KB   │              │  ├════╗ ← blocked by CSS
hero.jpg          180KB   │              ╠════════╝ ← found in HTML, parallel
font.woff2        28KB    │                      ├══════╗ ← found in CSS (late!)
                          │
                          └── render can begin ONLY after styles.css done
                                              ↑ FCP happens here

Problems visible:
1. font.woff2 discovered late (inside CSS) → late FCP text rendering
2. styles.css blocks render → nothing painted until ~300ms

Fix:
  Add: <link rel="preload" href="font.woff2" as="font" crossorigin>
  Add: inline critical CSS in <head>
```

### Waterfall Patterns to Recognize

```
Pattern 1 — Render-blocking chain (BAD):
  HTML → CSS → JS → render
  Each resource waits for the previous → sequential = slow

Pattern 2 — Parallel loading (GOOD):
  HTML → CSS ─┐
              ├─ render
          JS ─┘
  Resources load in parallel → faster

Pattern 3 — Critical resource buried in cascade (BAD):
  HTML → CSS → [font referenced in CSS] → render (with text)
  Font discovered only after CSS parsed → late discovery = FOUT

Pattern 4 — Preloaded critical resources (GOOD):
  HTML + <link rel="preload"> → CSS, font, hero-image → render
  Preload hints allow parallel loading from HTML → faster
```

---

## 11. Core Web Vitals Connection

CRP optimization directly impacts the three Core Web Vitals:

### LCP (Largest Contentful Paint) — Target: < 2.5s

```
LCP measures: when the largest visible content element is painted
Common LCP elements: hero image, large text block, video poster

CRP's impact on LCP:
  - If LCP element is above fold: it's on the critical path
  - CSS blocks rendering → delays LCP
  - Render-blocking scripts delay LCP
  - Large LCP image not preloaded → late discovery → slow LCP

LCP optimizations:
  ✅ Preload the LCP image: <link rel="preload" href="hero.jpg" as="image">
  ✅ Use fetchpriority="high" on the LCP image element
  ✅ Optimize LCP image: WebP, responsive sizes, correct dimensions
  ✅ Eliminate render-blocking resources above LCP element
  ✅ Minimize TTFB: CDN, server caching, edge delivery
```

### FCP (First Contentful Paint) — Target: < 1.8s

```
FCP measures: when first text/image content is painted

CRP's impact on FCP:
  - FCP = end of critical rendering path
  - Any render-blocking resource delays FCP
  - Large critical CSS delays FCP
  - TTFB delays FCP

FCP optimizations:
  ✅ Inline critical CSS (eliminate CSS network request)
  ✅ Reduce TTFB (CDN, server-side caching)
  ✅ Defer non-critical CSS
  ✅ Remove render-blocking scripts from <head>
  ✅ Use preconnect for external origins
```

### INP (Interaction to Next Paint) — Target: < 200ms

```
INP measures: responsiveness to user interactions
Less directly tied to CRP, but related:

CRP's indirect impact on INP:
  - Large JS bundles on critical path = more code to parse/execute
  - Parsing JS blocks main thread = delayed interactivity
  - Heavy initial JS = more GC pressure during user interaction

INP optimizations (CRP-adjacent):
  ✅ Code split: only load JS needed for current page
  ✅ Defer JS: let HTML render first, initialize JS after
  ✅ Reduce main thread work during load
```

---

## 12. Server-Side Optimizations

CRP optimization doesn't stop at the client. Server-side changes are often the highest leverage.

### TTFB — Time to First Byte

```
TTFB = time from browser sending request to receiving first byte of response

TTFB components:
  1. DNS resolution: 0–100ms
  2. TCP + TLS handshake: 0–200ms
  3. Server processing time: 0–500ms+ (the controllable one)
  4. Data transmission start: tiny

Reducing TTFB:
  ✅ CDN: route requests to nearest edge node (DNS + RTT)
  ✅ Server-side caching: cache rendered HTML (avoid DB queries on every request)
  ✅ Database query optimization: reduce server processing time
  ✅ Edge computing: run server logic at CDN edge nodes
  ✅ HTTP/2 + HTTPS: faster connection setup

Target TTFB: < 200ms (for good LCP)
```

### Compression

```
gzip vs Brotli compression:

File: 200KB uncompressed CSS

gzip -9:   ~60KB (70% reduction)  — universal support
brotli -9: ~50KB (75% reduction)  — modern browsers only

Use both: brotli for modern browsers, gzip as fallback
Brotli is especially effective for HTML and CSS (repetitive text)

Server header: Content-Encoding: br (Brotli) or gzip
Browser indicates support: Accept-Encoding: gzip, deflate, br
```

### Cache-Control for CRP Resources

```http
# HTML: short cache + revalidation
Cache-Control: no-cache
# No-cache = revalidate on every request (using ETag/Last-Modified)
# If content unchanged: server returns 304 Not Modified (no body)

# CSS/JS with content hash: long cache
Cache-Control: public, max-age=31536000, immutable
# /styles.abc123.css — content hash in filename
# Browser caches for 1 year — safe because URL changes when content changes

# Images: long cache
Cache-Control: public, max-age=86400
# 24 hours — balance freshness vs cache efficiency
```

---

## 13. Good Practices

### ✅ Inline critical CSS

```html
<head>
  <style>
    /* 3–10KB of critical above-fold CSS */
  </style>
  <link
    rel="preload"
    href="/styles.css"
    as="style"
    onload="this.rel='stylesheet'"
  />
</head>
```

### ✅ Use `defer` for non-critical scripts

```html
<!-- Scripts defer by default when type="module" -->
<script defer src="app.js"></script>
<script type="module" src="main.js"></script>
```

### ✅ Preload the LCP image

```html
<!-- Preload the image the browser will discover late (in CSS or JS) -->
<link rel="preload" href="/hero.webp" as="image" fetchpriority="high" />

<!-- Also: add fetchpriority="high" to the img tag itself -->
<img src="/hero.webp" fetchpriority="high" alt="Hero" />
```

### ✅ Use `preconnect` for external origins

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://api.example.com" />
```

### ✅ Serve with HTTP/2

```nginx
# Nginx: HTTP/2 configuration
server {
  listen 443 ssl http2;  # enable HTTP/2
  ssl_certificate /path/to/cert.pem;
  ssl_certificate_key /path/to/key.pem;
}
```

### ✅ Optimize images with modern formats

```html
<!-- WebP with fallback -->
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero" fetchpriority="high" />
</picture>
```

---

## 14. Bad Practices

### ❌ All CSS in one render-blocking file with no critical CSS inlining

```html
<!-- ❌ All 500KB of CSS blocks first paint -->
<link rel="stylesheet" href="everything.css" />
```

### ❌ Scripts in `<head>` without `async`/`defer`

```html
<!-- ❌ Blocks parsing and rendering until both scripts download and execute -->
<head>
  <script src="framework.js"></script>
  <!-- 200KB, 2 seconds download -->
  <script src="app.js"></script>
  <!-- depends on framework -->
</head>
```

### ❌ Not preloading the LCP resource

```html
<!-- ❌ LCP image discovered only when CSS is parsed — late -->
<!-- CSS rule: .hero { background-image: url('hero.jpg') } -->
<!-- Browser discovers hero.jpg only after downloading and parsing CSS -->
<!-- Add: <link rel="preload" href="hero.jpg" as="image"> -->
```

### ❌ Ignoring TTFB

```
❌ TTFB of 1200ms:
  1200ms just to start receiving bytes
  FCP cannot happen before TTFB + transfer + parse + render
  Any CRP optimization is moot if TTFB is 1.2 seconds

Fix: CDN, server caching, database optimization
Target: < 200ms TTFB
```

### ❌ Render-blocking fonts

```html
<!-- ❌ Fonts not preloaded — discovered only after CSS parses -->
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter" />

<!-- ✅ Preconnect to font origin + preload critical fonts -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link
  rel="preload"
  href="/fonts/inter-400.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

---

## 15. Common Mistakes

### Mistake 1 — Confusing DOMContentLoaded with the CRP completion

```javascript
// DOMContentLoaded fires when HTML is parsed + defer scripts executed
// This is NOT necessarily the same as first render

// First render can happen BEFORE DOMContentLoaded
// (if critical CSS is inlined, render doesn't wait for deferred scripts)

// It can also happen slightly AFTER
// (if CSSOM takes longer than DOM parsing)
```

### Mistake 2 — Preloading non-critical resources

```html
<!-- ❌ Preloading too aggressively — competes with critical resources -->
<link rel="preload" href="/page2-bundle.js" as="script" />
<!-- not needed for this page -->
<link rel="preload" href="/below-fold-image.jpg" as="image" />
<!-- not critical -->

<!-- Preload only resources on the actual critical path for the current page -->
```

### Mistake 3 — Forgetting `crossorigin` on font preloads

```html
<!-- ❌ Font preload without crossorigin — will be downloaded TWICE -->
<link rel="preload" href="/fonts/inter.woff2" as="font" />

<!-- ✅ Fonts are CORS resources — crossorigin attribute required -->
<link
  rel="preload"
  href="/fonts/inter.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

### Mistake 4 — Not checking real network waterfall

```
Many developers optimize based on localhost performance
(no network latency, instant file serving, no compression).

Always test CRP with:
  - Realistic throttling (DevTools → Network → Slow 3G / Fast 3G)
  - Real devices (Chrome DevTools → Remote Devices)
  - WebPageTest (https://webpagetest.org) — filmstrip and waterfall
  - Lighthouse → CRP diagnostic
```

---

## 16. Interview-Level Explanation

> **"What is the Critical Rendering Path and how do you optimize it?"**

**Strong answer:**

> "The Critical Rendering Path is the sequence of steps the browser must complete before rendering the first visible pixel: build the DOM from HTML, build the CSSOM from CSS, combine them into a render tree, run layout to compute geometry, paint pixels, and composite layers.
>
> What makes it 'critical' is that every step must complete before the browser can show anything. CSS is render-blocking by design — a single rule can affect any element, so the browser waits for all CSS before rendering. JavaScript is even more severe — a script tag without `async` or `defer` stops both HTML parsing and rendering until the script downloads and executes.
>
> The core optimization strategies are: inline critical CSS to eliminate the CSS network request from the critical path; use `defer` or move scripts to the bottom of `<body>` so they don't block parsing; preload the LCP image with `<link rel="preload" fetchpriority="high">` since it's often buried in CSS and discovered late; use `preconnect` to establish connections early to external origins; and reduce TTFB through CDN and server caching.
>
> The CRP directly drives Core Web Vitals. LCP measures when the largest visible element paints — if it's a hero image, preloading it is critical. FCP measures the first contentful paint — inline critical CSS and defer everything else. INP is less directly CRP-related but benefits from code splitting so less JavaScript is on the critical path.
>
> Measurement-first is essential here: use the Navigation Timing API for precise CRP milestone timing, the Paint Timing API for FCP, and the network waterfall in DevTools to visualize the dependency chain. The most common CRP problem is a deep dependency chain — where resource B can't start until A finishes — and `preload` hints break that chain by letting the browser discover B before A."

---

## 17. Exercises

### Exercise 1 — Identify CRP bottlenecks

Given this HTML, identify every CRP bottleneck and calculate the minimum theoretical time to first render (assume 100ms RTT, 1MB/s bandwidth):

```html
<!DOCTYPE html>
<html>
  <head>
    <link rel="stylesheet" href="reset.css" />
    <!-- 5KB -->
    <link rel="stylesheet" href="main.css" />
    <!-- 80KB -->
    <script src="jquery.js"></script>
    <!-- 90KB -->
    <script src="app.js"></script>
    <!-- 40KB -->
  </head>
  <body>
    <h1>Hello World</h1>
  </body>
</html>
```

<details>
<summary>Analysis + Fix</summary>

```
CRP Bottlenecks:

1. Scripts in <head> without async/defer:
   - jquery.js (90KB): parser-blocking AND render-blocking
   - app.js (40KB): also parser-blocking

2. Multiple CSS files without critical CSS inlining:
   - reset.css: render-blocking (though small)
   - main.css: render-blocking (80KB = significant)

3. No preload hints: resources discovered sequentially

Minimum time to first render (at 100ms RTT, 1MB/s):
  HTML download:       15KB ÷ 1MB/s = 15ms
  HTML parse:          ~5ms
  CSS requests:        100ms RTT + sequential
    reset.css (5KB):   100ms + 5ms = 105ms
    main.css (80KB):   already requested: 100ms + 80ms = 180ms
  Script requests (SEQUENTIAL because parser-blocking):
    After CSS loads (scripts wait for CSS due to inline scripts):
    jquery.js (90KB): 100ms + 90ms = 190ms
    app.js (40KB):    100ms + 40ms = 140ms (waits for jquery)

  Total: HTML (20ms) + CSS parallel (~180ms) +
         wait for CSS + jquery (190ms) + app.js (140ms)

  Approximately: ~550ms minimum theoretical FCP

FIXED version:
  <head>
    <style>/* critical CSS inlined — 10KB */</style>
    <link rel="preload" href="main.css" as="style" onload="this.rel='stylesheet'">
    <script defer src="jquery.js"></script>
    <script defer src="app.js"></script>
  </head>

  Fixed minimum FCP:
    HTML + inlined critical CSS: 15ms + ~5ms = ~20ms
    Render can begin immediately after HTML + critical CSS parsed!
    FCP: ~20–30ms (just HTML + inlined styles)

  Improvement: 550ms → 30ms = 18× faster FCP
```

</details>

---

### Exercise 2 — Write a CRP measurement script

Write a JavaScript function that measures and reports all critical CRP milestones using the Performance API:

```javascript
function measureCRP() {
  // Should output:
  // DNS: Xms
  // TCP Connect: Xms
  // TTFB: Xms
  // HTML Download: Xms
  // DOM Interactive: Xms (HTML parsed)
  // DOMContentLoaded: Xms
  // FCP: Xms
  // LCP: Xms
  // Load: Xms
}
```

<details>
<summary>Solution</summary>

```javascript
async function measureCRP() {
  const [nav] = performance.getEntriesByType("navigation");

  // Navigation timing (synchronous measurements)
  const timings = {
    DNS: nav.domainLookupEnd - nav.domainLookupStart,
    "TCP Connect": nav.connectEnd - nav.connectStart,
    TLS:
      nav.connectEnd > nav.secureConnectionStart
        ? nav.connectEnd - nav.secureConnectionStart
        : 0,
    TTFB: nav.responseStart - nav.requestStart,
    "HTML Download": nav.responseEnd - nav.responseStart,
    "DOM Interactive": nav.domInteractive,
    DOMContentLoaded: nav.domContentLoadedEventEnd,
    Load: nav.loadEventEnd,
  };

  // Paint timing (may need observer)
  const paintEntries = performance.getEntriesByType("paint");
  paintEntries.forEach((entry) => {
    timings[entry.name] = entry.startTime;
  });

  // LCP (need observer — may not be finalized yet)
  await new Promise((resolve) => {
    const lcpObserver = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lcp = entries[entries.length - 1];
      timings["LCP"] = lcp.startTime;
      lcpObserver.disconnect();
      resolve();
    });
    lcpObserver.observe({ entryTypes: ["largest-contentful-paint"] });

    // Fallback: resolve after timeout
    setTimeout(resolve, 3000);
  });

  // Format and display
  console.group("Critical Rendering Path Metrics");
  Object.entries(timings)
    .sort(([, a], [, b]) => a - b)
    .forEach(([name, ms]) => {
      const bar = "█".repeat(Math.floor(ms / 20));
      const status = ms < 100 ? "✅" : ms < 500 ? "⚠️" : "❌";
      console.log(
        `${status} ${name.padEnd(20)} ${ms.toFixed(0).padStart(6)}ms ${bar}`,
      );
    });
  console.groupEnd();

  return timings;
}

// Call after page load
window.addEventListener("load", () => {
  setTimeout(measureCRP, 100); // slight delay for LCP to finalize
});
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](./01-rendering-pipeline.md) — Full pipeline after CRP completes
- [`browser-internals/02-dom-tree-creation.md`](./02-dom-tree-creation.md) — DOM construction in depth
- [`browser-internals/03-cssom.md`](./03-cssom.md) — CSSOM construction and render-blocking
- [`browser-internals/09-browser-caching.md`](./09-browser-caching.md) — Caching to skip CRP on repeat visits
- [`browser-internals/10-ssr-csr-isr-streaming.md`](./10-ssr-csr-isr-streaming.md) — SSR to optimize CRP server-side

---

<div align="center">

**Next:** [`browser-internals/09-browser-caching.md`](./09-browser-caching.md) →

</div>
