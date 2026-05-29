# 08 — Bundle Optimization

> **"Your bundle is a contract with your users: every byte you ship, they must download, parse, and execute. The fastest code is the code they never receive."**

Bundle optimization is the discipline of reducing the JavaScript, CSS, and asset payload that reaches the browser. It directly impacts First Contentful Paint, Time to Interactive, and Interaction to Next Paint — the metrics that determine whether users stay or leave. This document covers the full optimization toolkit: tree shaking, code splitting, lazy loading, compression, differential serving, and the analysis tools that tell you where your bytes are going.

---

## 📚 Table of Contents

1. [Why Bundle Size Matters](#1-why-bundle-size-matters)
2. [Analyzing Your Bundle](#2-analyzing-your-bundle)
3. [Tree Shaking — Eliminating Dead Code](#3-tree-shaking--eliminating-dead-code)
4. [Code Splitting — Loading What You Need](#4-code-splitting--loading-what-you-need)
5. [Dynamic Imports](#5-dynamic-imports)
6. [Route-Based Code Splitting](#6-route-based-code-splitting)
7. [Lazy Loading Components](#7-lazy-loading-components)
8. [Dependency Auditing](#8-dependency-auditing)
9. [Replacing Heavy Dependencies](#9-replacing-heavy-dependencies)
10. [Compression — Gzip and Brotli](#10-compression--gzip-and-brotli)
11. [Differential Serving](#11-differential-serving)
12. [CSS Optimization](#12-css-optimization)
13. [Image Optimization](#13-image-optimization)
14. [Caching Strategy for Bundles](#14-caching-strategy-for-bundles)
15. [Build Configuration — Webpack and Vite](#15-build-configuration--webpack-and-vite)
16. [Good Practices](#16-good-practices)
17. [Bad Practices](#17-bad-practices)
18. [Common Mistakes](#18-common-mistakes)
19. [Interview-Level Explanation](#19-interview-level-explanation)
20. [Exercises](#20-exercises)

---

## 1. Why Bundle Size Matters

JavaScript parsing and execution are among the most expensive operations a browser performs on page load:

```
Cost of 1MB JavaScript bundle on a mid-range mobile device:

Download (3G, 1.6Mbps):  ~5 seconds
Parse:                    ~500ms (CPU: tokenize JS into AST)
Compile:                  ~300ms (JIT: AST to bytecode)
Execute:                  ~200ms (run top-level code)
──────────────────────────────────
Total:                    ~6 seconds before any content is interactive

vs. 1MB image:
Download (3G):            ~5 seconds
Decode:                   ~50ms
Display:                  ~10ms
──────────────────────────────────
Total:                    ~5.06 seconds — no parse/compile/execute
```

**JavaScript is the most expensive byte you can ship** because it blocks interactivity.

### The Performance Budget Mindset

```
Rule of thumb performance budgets:
  Total JS (compressed):       < 200KB for fast TTI on 3G
  Total CSS (compressed):      < 50KB
  Initial JS (critical path):  < 100KB

Above these: users on slow connections experience meaningful delays
Every 10KB of JS removed: ~30-50ms faster on mid-range mobile
```

---

## 2. Analyzing Your Bundle

You can't optimize what you can't see. Always start with analysis.

### Webpack Bundle Analyzer

```bash
# Install
npm install --save-dev webpack-bundle-analyzer

# Add to webpack config
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',      // generates HTML report
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,         // don't auto-open
      generateStatsFile: true,     // save stats.json for further analysis
    }),
  ],
};

# Or run standalone:
npx webpack --json > stats.json
npx webpack-bundle-analyzer stats.json
```

### Vite Bundle Analysis

```bash
# rollup-plugin-visualizer
npm install --save-dev rollup-plugin-visualizer

# vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      filename:  'bundle-report.html',
      open:      true,
      gzipSize:  true,   // show gzip sizes
      brotliSize: true,  // show brotli sizes
    }),
  ],
});

npm run build
# Opens bundle-report.html in browser
```

### What to Look For in the Report

```
Treemap view interpretation:
  Large rectangles = large modules = most impactful to optimize

Common offenders:
  lodash               → 70KB raw (use lodash-es + tree shaking or individual imports)
  moment.js            → 232KB raw (replace with date-fns, dayjs, or Temporal API)
  chart.js             → 60KB (often bundled fully even if using one chart type)
  @mui/icons-material  → bundles ALL icons if imported wrongly
  firebase             → massive if fully imported (use modular SDK)
  rxjs                 → large if barrel imports used

Questions to ask:
  "Why is lodash 70KB when we only use 3 functions?"
  "Why are we bundling ALL moment.js locales?"
  "Why is this vendor chunk 1.2MB?"
```

### Source Map Explorer

```bash
npm install --save-dev source-map-explorer

# Build with source maps
npm run build

# Analyze
npx source-map-explorer dist/main.*.js
# Shows breakdown of which source files contribute to bundle size
```

---

## 3. Tree Shaking — Eliminating Dead Code

Tree shaking removes unused exports from the final bundle. It requires **static ES module syntax** — the bundler must be able to determine at build time which exports are used.

### Why Tree Shaking Requires ES Modules

```javascript
// ❌ CommonJS — dynamic, can't tree-shake
const utils = require("./utils");
utils.onlyThisFunction(); // bundler can't know at build time which parts are used

// ✅ ES Modules — static, tree-shakeable
import { onlyThisFunction } from "./utils";
// bundler knows: only `onlyThisFunction` is imported → rest can be removed
```

### Enabling Tree Shaking

```javascript
// webpack.config.js
module.exports = {
  mode: "production", // automatically enables tree shaking
  optimization: {
    usedExports: true, // marks unused exports for removal
    sideEffects: false, // enables more aggressive tree shaking
  },
};
```

### `sideEffects` in package.json

```json
// package.json — your library
{
  "name": "my-library",
  "sideEffects": false
  // Tells bundlers: importing from this package has no side effects
  // Unused exports can be safely removed
}

// Or: specify files that DO have side effects
{
  "sideEffects": [
    "./src/styles.css",     // CSS imports have side effects
    "./src/polyfills.js"    // Polyfills modify globals
  ]
}
```

### Common Tree-Shaking Failures

```javascript
// ❌ Barrel files import everything
// src/components/index.ts:
export { Button } from "./Button";
export { Modal } from "./Modal";
export { Dropdown } from "./Dropdown";
// ... 50 more exports

// src/App.tsx:
import { Button } from "./components"; // imports the barrel
// Even though only Button is used, bundler may pull in all 50 components
// if any of them have side effects

// ✅ Import directly (bypass barrel)
import { Button } from "./components/Button";
```

```javascript
// ❌ Lodash barrel import — pulls in ALL of lodash
import { debounce, throttle } from "lodash";
// entire lodash included (70KB)

// ✅ Lodash-es with tree shaking
import { debounce, throttle } from "lodash-es";
// only debounce and throttle included (~2KB)

// ✅ Or: individual function imports (works with CJS lodash)
import debounce from "lodash/debounce";
import throttle from "lodash/throttle";
```

---

## 4. Code Splitting — Loading What You Need

Code splitting divides your JavaScript into separate chunks that are loaded on demand. Instead of one massive bundle, users load only what they need for the current page.

```
Without code splitting:
  main.bundle.js: 1.2MB
  Contains: auth, dashboard, settings, reports, admin, charting, editor
  User visits /login: downloads all 1.2MB before anything is interactive

With code splitting:
  main.js: 80KB (core app shell)
  auth.chunk.js: 20KB
  dashboard.chunk.js: 150KB
  settings.chunk.js: 60KB
  reports.chunk.js: 300KB (heavy — charting library)
  admin.chunk.js: 100KB

  User visits /login: downloads main.js (80KB) + auth.chunk.js (20KB) = 100KB
  100KB vs 1.2MB = 12× less code to parse before interactive
```

### Webpack Automatic Splitting

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: "all", // split async AND sync chunks
      minSize: 20_000, // only split if > 20KB
      maxSize: 250_000, // target max 250KB per chunk
      cacheGroups: {
        // Vendor chunk: rarely-changing third-party libraries
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          priority: 10,
        },
        // Separate chunk for react + react-dom (always reused)
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: "react",
          priority: 20,
        },
      },
    },
  },
};
```

---

## 5. Dynamic Imports

`import()` is the mechanism that enables code splitting. It loads a module asynchronously and returns a Promise.

```javascript
// Static import (always included in bundle):
import { heavyChart } from "./charting";

// Dynamic import (separate chunk, loaded on demand):
const { heavyChart } = await import("./charting");
// → creates a separate charting.chunk.js file
// → only loaded when this code runs
```

### Preloading and Prefetching Chunks

```javascript
// Hint to browser: load this chunk when browser is idle (prefetch)
const Chart = await import(
  /* webpackPrefetch: true */
  "./components/Chart"
);

// Hint: high-priority, load ASAP (preload — for chunks you know you'll need soon)
const Dashboard = await import(
  /* webpackPreload: true */
  "./pages/Dashboard"
);
```

Webpack transforms these into:

```html
<!-- prefetch: loaded during idle time -->
<link rel="prefetch" href="/chunks/chart.abc123.js" />

<!-- preload: loaded with high priority -->
<link rel="preload" href="/chunks/dashboard.abc123.js" as="script" />
```

---

## 6. Route-Based Code Splitting

The most impactful code splitting strategy: one chunk per route.

### React (with React.lazy)

```javascript
// router.tsx
import React, { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

// Lazy-load each page — creates separate chunks
const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));
const Reports = lazy(() => import("./pages/Reports"));
const Admin = lazy(() => import("./pages/Admin"));

function AppRouter() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/reports" element={<Reports />} />
        <Route path="/admin" element={<Admin />} />
      </Routes>
    </Suspense>
  );
}

// Result: separate chunk per page
// User on /dashboard: downloads main.js + dashboard.chunk.js
// Admin chunk only downloaded when user navigates to /admin
```

### Next.js (Automatic Route Splitting)

```javascript
// Next.js automatically code-splits by page file
// pages/dashboard.tsx → dashboard bundle (automatic)
// pages/reports.tsx → reports bundle (automatic)
// No manual lazy() needed

// For dynamic imports within a page:
import dynamic from "next/dynamic";

const HeavyChart = dynamic(() => import("../components/HeavyChart"), {
  loading: () => <ChartSkeleton />,
  ssr: false, // disable SSR for client-only components
});
```

### Vite Route Splitting

```javascript
// vite + react-router
// vite automatically splits dynamic imports into separate chunks
const router = createBrowserRouter([
  {
    path: "/",
    element: <Layout />,
    children: [
      {
        path: "dashboard",
        lazy: async () => {
          const { Dashboard } = await import("./pages/Dashboard");
          return { Component: Dashboard };
        },
      },
      {
        path: "reports",
        lazy: async () => {
          const { Reports } = await import("./pages/Reports");
          return { Component: Reports };
        },
      },
    ],
  },
]);
```

---

## 7. Lazy Loading Components

Beyond routes, individual heavy components within a page can be lazy-loaded.

### Lazy Loading Modals and Dialogs

```javascript
// ✅ Modal code only loads when user opens it
import { lazy, Suspense, useState } from "react";

const PaymentModal = lazy(() => import("./PaymentModal"));

function Checkout() {
  const [showModal, setShowModal] = useState(false);

  return (
    <>
      <button onClick={() => setShowModal(true)}>Pay Now</button>

      {showModal && (
        <Suspense fallback={<ModalSkeleton />}>
          <PaymentModal onClose={() => setShowModal(false)} />
        </Suspense>
      )}
    </>
  );
}
// PaymentModal chunk: only downloaded when user clicks "Pay Now"
```

### Lazy Loading Below-Fold Components

```javascript
// Load heavy components only when they scroll into view
import { lazy, Suspense } from "react";
import { useIntersectionObserver } from "./hooks/useIntersectionObserver";

const HeavyChart = lazy(() => import("./HeavyChart"));

function AnalyticsSection() {
  const { ref, isVisible } = useIntersectionObserver({ rootMargin: "200px" });

  return (
    <section ref={ref}>
      {isVisible ? (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      ) : (
        <ChartSkeleton />
      )}
    </section>
  );
}
```

---

## 8. Dependency Auditing

Third-party dependencies are often the biggest contributor to bundle bloat.

### Finding Large Dependencies

```bash
# Install cost-of-modules tool
npm install -g cost-of-modules

# In your project:
cost-of-modules
# Shows: each dependency, its size, and what percentage of total it is

# Or: use bundlephobia.com
# Go to: https://bundlephobia.com
# Search: moment, lodash, firebase, etc.
# Shows: min size, min+gzip size, download time
```

### Auditing with package.json

```bash
# Find duplicate packages (multiple versions in node_modules):
npx npm-dedupe

# Check for outdated packages (smaller versions may exist):
npm outdated

# Find packages used only in one place (may be inlineable):
npx depcheck
```

### Common Offenders and Their Sizes

| Package                     | Raw   | Gzip | Better Alternative                         |
| --------------------------- | ----- | ---- | ------------------------------------------ |
| `moment`                    | 232KB | 66KB | `date-fns` (tree-shakeable), `dayjs` (7KB) |
| `lodash`                    | 72KB  | 25KB | `lodash-es` + tree shaking, or native      |
| `axios`                     | 45KB  | 14KB | native `fetch`, `ky` (4KB)                 |
| `jquery`                    | 87KB  | 30KB | native DOM APIs                            |
| `react-icons` (all)         | 40MB  | —    | import individual icons                    |
| `@mui/icons-material` (all) | ~6MB  | —    | import path-specific                       |
| `validator.js`              | 47KB  | 15KB | individual validation functions            |
| `bluebird`                  | 83KB  | 27KB | native Promises                            |

---

## 9. Replacing Heavy Dependencies

### Moment.js → date-fns

```javascript
// ❌ moment.js: 232KB, imports all locales
import moment from "moment";
const formatted = moment(date).format("MMM D, YYYY");

// ✅ date-fns: tree-shakeable, ~2KB for format
import { format } from "date-fns";
const formatted = format(date, "MMM d, yyyy");
```

### Lodash → Native + Lodash-es

```javascript
// ❌ Lodash barrel import: 70KB
import { debounce, cloneDeep, groupBy, sortBy } from "lodash";

// ✅ Lodash-es: only used functions (~5KB total)
import { debounce, cloneDeep, groupBy, sortBy } from "lodash-es";

// ✅✅ Native alternatives: 0KB
// debounce → custom implementation (~300 bytes)
// cloneDeep → structuredClone (native, modern browsers)
// groupBy → Object.groupBy (native, 2024+) or reduce
// sortBy → Array.sort with comparator

const grouped = Object.groupBy(items, (item) => item.category);
const deep = structuredClone(obj);
```

### Replacing React Icon Imports

```javascript
// ❌ Named import from top-level: imports ALL icons (6MB uncompressed)
import { FaHome, FaUser } from "react-icons/fa";

// ✅ Path-specific import: only the icons you use
import FaHome from "react-icons/fa/FaHome";
import FaUser from "react-icons/fa/FaUser";

// Or with tree shaking (works if package supports it):
import { Home, User } from "lucide-react"; // tree-shakeable by design
```

---

## 10. Compression — Gzip and Brotli

Compression dramatically reduces transfer size for text-based assets.

### Compression Results

```
main.js:         1,200KB raw
                   380KB gzip (68% reduction)
                   320KB brotli (73% reduction)

main.css:          150KB raw
                    30KB gzip (80% reduction)
                    25KB brotli (83% reduction)

Total savings (brotli vs raw): ~780KB saved on every page load
```

### Server Configuration

```nginx
# Nginx: serve pre-compressed files
server {
  gzip_static on;    # serve .gz files if they exist
  brotli_static on;  # serve .br files if they exist

  # Dynamic compression fallback
  gzip on;
  gzip_types text/plain text/css application/javascript application/json;
  gzip_comp_level 6;

  brotli on;
  brotli_types text/plain text/css application/javascript application/json;
  brotli_comp_level 6;
}
```

### Build-Time Compression

```javascript
// vite.config.ts — pre-compress during build
import compression from "vite-plugin-compression";

export default defineConfig({
  plugins: [
    compression({
      algorithm: "gzip",
      ext: ".gz",
    }),
    compression({
      algorithm: "brotliCompress",
      ext: ".br",
    }),
  ],
});
```

```javascript
// webpack.config.js
const CompressionPlugin = require("compression-webpack-plugin");
const zlib = require("zlib");

module.exports = {
  plugins: [
    new CompressionPlugin({ algorithm: "gzip" }),
    new CompressionPlugin({
      filename: "[path][base].br",
      algorithm: "brotliCompress",
      test: /\.(js|css|html|svg)$/,
      compressionOptions: {
        params: { [zlib.constants.BROTLI_PARAM_QUALITY]: 11 },
      },
    }),
  ],
};
```

---

## 11. Differential Serving

Serve modern JavaScript to modern browsers (smaller bundles) and legacy JavaScript only to browsers that need it.

### The Problem with Transpilation

```javascript
// Source: modern JavaScript
const user = users.find((u) => u.id === id);
const names = users.map(({ name }) => name);

// Transpiled for IE11 (unnecessary for 99%+ of users in 2024):
("use strict");
var user = users.find(function (u) {
  return u.id === id;
});
var names = users.map(function (_ref) {
  var name = _ref.name;
  return name;
});

// Transpilation cost: ~30% larger bundle, includes core-js polyfills
```

### Module/NoModule Pattern

```html
<!-- Modern browsers: load ES modules (smaller, untranspiled) -->
<script type="module" src="/app.modern.js"></script>

<!-- Legacy browsers: load transpiled bundle (older syntax, polyfills) -->
<script nomodule src="/app.legacy.js"></script>

<!-- Modern browsers: ignore nomodule scripts
     Legacy browsers: ignore type="module" scripts -->
```

### Vite Differential Serving

```javascript
// vite.config.ts
export default defineConfig({
  build: {
    target: 'es2020',    // modern browsers only
    // Output: uses modern syntax, no unnecessary polyfills
  },
});

// For legacy support (separate legacy build):
import legacy from '@vitejs/plugin-legacy';

export default defineConfig({
  plugins: [
    legacy({
      targets: ['defaults', 'not IE 11'], // specify browsers
      modernPolyfills: true,
    }),
  ],
});
```

---

## 12. CSS Optimization

### Purging Unused CSS

```javascript
// vite.config.ts with Tailwind — purging is automatic
// tailwind.config.js
module.exports = {
  content: ["./src/**/*.{ts,tsx,js,jsx,html}"],
  // Tailwind scans these files and removes unused utilities
};

// Custom CSS purging with PurgeCSS:
import purgecss from "@fullhuman/postcss-purgecss";

export default {
  plugins: [
    purgecss({
      content: ["./src/**/*.{html,tsx,ts}"],
      defaultExtractor: (content) => content.match(/[\w-/:]+(?<!:)/g) || [],
    }),
  ],
};
```

### CSS Minification

```javascript
// vite: built-in with lightningcss
export default defineConfig({
  css: {
    transformer: "lightningcss",
    lightningcss: {
      minify: true,
    },
  },
});

// webpack: css-minimizer-webpack-plugin
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");
module.exports = {
  optimization: {
    minimizer: ["...", new CssMinimizerPlugin()],
  },
};
```

### Critical CSS Inlining

```javascript
// Inline above-fold CSS to eliminate render-blocking request
// See: browser-internals/08-critical-rendering-path.md

// critters: Webpack plugin for automatic critical CSS inlining
const Critters = require("critters-webpack-plugin");
module.exports = {
  plugins: [new Critters()],
};
```

---

## 13. Image Optimization

Images are often the largest contributors to page weight.

### Format Selection

```
WebP:  30-40% smaller than JPEG/PNG, near-universal support (96%+)
AVIF:  50-55% smaller than JPEG, growing support (88%+)
SVG:   ideal for icons and illustrations — scales perfectly

Usage: AVIF with WebP fallback with JPEG/PNG fallback

<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Description" loading="lazy">
</picture>
```

### Responsive Images

```html
<!-- Deliver appropriate size for each device -->
<img
  src="/hero-800.jpg"
  srcset="
    /hero-400.jpg   400w,
    /hero-800.jpg   800w,
    /hero-1200.jpg 1200w,
    /hero-1600.jpg 1600w
  "
  sizes="(max-width: 400px) 400px,
         (max-width: 800px) 800px,
         (max-width: 1200px) 1200px,
         1600px"
  alt="Hero image"
  loading="lazy"
  decoding="async"
/>
```

### Build-Time Image Optimization

```javascript
// vite-imagetools
import { imagetools } from "vite-imagetools";

export default defineConfig({
  plugins: [imagetools()],
});

// In component:
import heroUrl from "./hero.jpg?format=webp&width=1200";
// Outputs: optimized WebP at 1200px wide
```

---

## 14. Caching Strategy for Bundles

Optimal caching uses content hashes to enable long-term caching of static assets.

```javascript
// webpack.config.js — content hashing
module.exports = {
  output: {
    filename: "[name].[contenthash:8].js", // app code: changes with content
    chunkFilename: "[name].[contenthash:8].chunk.js",
  },
  optimization: {
    // Separate runtime chunk (rarely changes — cache separately)
    runtimeChunk: "single",
    // Stable module IDs (don't change on unrelated module additions)
    moduleIds: "deterministic",
    chunkIds: "deterministic",
  },
};
```

### Chunking for Cache Efficiency

```javascript
// Goal: large stable dependencies → separate chunks → cached long-term
// Frequently changing app code → separate from dependencies

// Vendor split: node_modules rarely change → separate cache entry
splitChunks: {
  cacheGroups: {
    // React: never changes between deployments
    react: {
      test:     /[\\/]node_modules[\\/](react|react-dom|react-router)[\\/]/,
      name:     'react-vendor',
      chunks:   'all',
      priority: 30,
    },
    // Other vendors: change occasionally
    vendors: {
      test:     /[\\/]node_modules[\\/]/,
      name:     'vendors',
      chunks:   'all',
      priority: 20,
      maxSize:  250_000, // split large vendor chunks
    },
  },
},
```

### Cache Headers for Bundles

```nginx
# Hashed assets: cache forever (URL changes when content changes)
location ~* \.(js|css|woff2|png|jpg|webp)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}

# HTML and service worker: never cache
location ~* \.(html|json)$ {
  add_header Cache-Control "no-cache, must-revalidate";
}
```

---

## 15. Build Configuration — Webpack and Vite

### Webpack Production Configuration

```javascript
// webpack.config.js (production)
const path = require("path");
const TerserPlugin = require("terser-webpack-plugin");

module.exports = {
  mode: "production",

  entry: { app: "./src/index.tsx" },

  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].[contenthash:8].js",
    chunkFilename: "[name].[contenthash:8].chunk.js",
    clean: true,
    publicPath: "auto",
  },

  optimization: {
    minimizer: [
      new TerserPlugin({
        parallel: true,
        terserOptions: {
          compress: { drop_console: true }, // remove console.log in prod
        },
      }),
    ],
    splitChunks: {
      chunks: "all",
      maxSize: 250_000,
    },
    runtimeChunk: "single",
    moduleIds: "deterministic",
    usedExports: true, // tree shaking
    sideEffects: true, // respect package.json sideEffects
  },

  module: {
    rules: [
      {
        test: /\.[jt]sx?$/,
        exclude: /node_modules/,
        use: ["babel-loader"],
      },
    ],
  },

  resolve: {
    extensions: [".tsx", ".ts", ".js"],
    alias: {
      "@": path.resolve(__dirname, "src"),
    },
  },
};
```

### Vite Production Configuration

```typescript
// vite.config.ts (production)
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],

  build: {
    target: "es2020",
    minify: "terser", // or 'esbuild' (faster, less aggressive)
    sourcemap: false, // disable in production (or 'hidden')
    cssMinify: true,
    terserOptions: {
      compress: { drop_console: true },
    },
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          // React ecosystem: separate chunk, rarely changes
          if (
            id.includes("node_modules/react") ||
            id.includes("node_modules/react-dom")
          ) {
            return "react-vendor";
          }
          // Large libraries: separate chunks
          if (id.includes("node_modules/recharts")) {
            return "charts-vendor";
          }
          // Everything else in node_modules: vendors chunk
          if (id.includes("node_modules")) {
            return "vendors";
          }
        },
        chunkFileNames: "[name]-[hash].js",
        assetFileNames: "[name]-[hash].[ext]",
      },
    },
    // Warn when any chunk exceeds 500KB
    chunkSizeWarningLimit: 500,
  },
});
```

---

## 16. Good Practices

### ✅ Set a performance budget and enforce it in CI

```javascript
// webpack.config.js
module.exports = {
  performance: {
    maxAssetSize: 250_000, // warn on assets > 250KB
    maxEntrypointSize: 500_000, // error on entry > 500KB
    hints: process.env.CI ? "error" : "warning", // fail CI on budget violation
  },
};
```

### ✅ Analyze bundle on every significant dependency change

```bash
# package.json scripts
"analyze": "ANALYZE=true npm run build"
# Run after: npm install new-large-library
```

### ✅ Import specific functions, not entire libraries

```javascript
// ✅ Only the formatDistance function
import { formatDistance } from "date-fns";

// ❌ Entire date-fns library
import dateFns from "date-fns";
```

### ✅ Use native APIs where possible

```javascript
// ✅ Native (0KB) vs library (~5KB)
const sorted = [...arr].sort((a, b) => a - b); // native
const cloned = structuredClone(obj); // native
const groups = Object.groupBy(items, (i) => i.type); // native (2024+)
```

---

## 17. Bad Practices

### ❌ Importing full libraries when only using a fraction

```javascript
// ❌ Importing all of Firebase (500KB+)
import firebase from "firebase/app";
import "firebase/auth";
import "firebase/firestore";

// ✅ Modular Firebase SDK (tree-shakeable)
import { initializeApp } from "firebase/app";
import { getAuth, signInWithEmailAndPassword } from "firebase/auth";
import { getFirestore, doc, getDoc } from "firebase/firestore";
```

### ❌ Using `require()` in ES module context

```javascript
// ❌ Dynamic require — prevents tree shaking
const { format } = require("date-fns");

// ✅ Static import — enables tree shaking
import { format } from "date-fns";
```

### ❌ Including development code in production bundles

```javascript
// ❌ Console.log and debug utilities in production
if (process.env.NODE_ENV === "development") {
  console.log("Debug:", data); // not removed if build doesn't replace NODE_ENV
}

// ✅ Ensure NODE_ENV is replaced at build time
// webpack: DefinePlugin({ 'process.env.NODE_ENV': JSON.stringify('production') })
// vite: automatic
```

---

## 18. Common Mistakes

### Mistake 1 — Lazy loading too aggressively

```javascript
// ❌ Lazy loading a tiny component — loading state worse than the savings
const Spinner = lazy(() => import("./Spinner")); // 0.5KB component
// User sees loading flicker for a component that's faster to just bundle

// ✅ Only lazy load components > ~20KB or that are rarely used
const HeavyDataGrid = lazy(() => import("./HeavyDataGrid")); // 150KB
```

### Mistake 2 — Vendor chunk too large

```javascript
// ❌ Single vendor chunk: 1.2MB
// Everything in node_modules → one chunk
// Any dependency update invalidates the entire vendor cache

// ✅ Split vendors strategically
// react/react-dom → react-vendor (5KB often, cache for months)
// charting library → charts-vendor (200KB, rarely changes)
// other deps → vendors (changes occasionally)
```

### Mistake 3 — Not removing source maps in production

```javascript
// ❌ Source maps served to users — exposes source code
devtool: "source-map";
// Users can see your original source code in DevTools!
// Also: sourcemaps add to bundle size served

// ✅ Hidden source maps: uploaded to error monitoring, not served to users
devtool: "hidden-source-map";
// Or: 'nosources-source-map' — stack traces but no source
// Or: false — no source maps at all
```

---

## 19. Interview-Level Explanation

> **"How do you optimize JavaScript bundle size? What tools and techniques do you use?"**

**Strong answer:**

> "Bundle optimization starts with measurement — you can't optimize what you haven't analyzed. I use webpack-bundle-analyzer or rollup-plugin-visualizer to get a treemap of what's in the bundle. The first thing I look for is large unexpected dependencies: a project I optimized had moment.js (232KB) pulled in by a seemingly innocent date utility, and replacing it with date-fns cut 80KB from the bundle.
>
> Tree shaking removes unused code — but it requires static ES module imports and the packages need to declare `sideEffects: false` in their package.json. The biggest tree-shaking win is often switching from barrel imports to direct imports. `import { Button } from './components'` may pull in all 50 components through the barrel file; `import { Button } from './components/Button'` only pulls in what you need.
>
> Code splitting divides the bundle into chunks loaded on demand. Route-based splitting is the highest-leverage approach — if a user visits /login, they don't need the reports page or admin panel. With React's `lazy()` and Suspense, each page becomes its own chunk. The initial bundle shrinks dramatically; each additional route loads only when navigated to.
>
> For compression, Brotli achieves 70-80% reduction on JavaScript — a 1MB bundle becomes 250-320KB. Serving pre-compressed files from the CDN is essentially free performance.
>
> For long-term caching, content hashing in the filename — like `app.abc123.js` — means assets can be cached indefinitely. The filename changes only when content changes, so new deployments don't invalidate the cache for unchanged modules. Separating vendor dependencies (react, react-dom) from app code means users don't need to re-download React every time you ship a feature.
>
> The discipline is: measure first, identify the biggest contributors, fix the highest-impact ones, measure again. Adding `useMemo` everywhere and lazy-loading every component is premature optimization — but replacing moment with dayjs, fixing a bad barrel import, and adding route splitting can often cut initial load time in half."

---

## 20. Exercises

### Exercise 1 — Bundle analysis

Given the following bundle analyzer output, identify the top 3 optimizations and estimate their potential savings:

```
Bundle composition:
  vendors.js (850KB):
    moment: 232KB
    lodash: 71KB
    axios: 45KB
    jquery: 87KB
    react + react-dom: 140KB
    other deps: 275KB

  app.js (420KB):
    pages/Dashboard: 180KB (includes recharts: 160KB)
    pages/Settings: 40KB
    pages/Profile: 35KB
    pages/Reports: 90KB (also includes recharts: 160KB — DUPLICATE!)
    shared components: 75KB
```

<details>
<summary>Answer</summary>

```
Top 3 optimizations:

1. Replace moment → dayjs (saves ~225KB after tree shaking)
   moment: 232KB → dayjs: 7KB = 225KB saved
   Action: npm install dayjs, update imports

2. Fix duplicate recharts (saves ~160KB from one of the pages)
   recharts appears in both Dashboard (160KB) and Reports (160KB)
   Should be in shared vendors chunk, not duplicated
   Action: configure splitChunks to extract recharts into its own chunk
   Savings: 160KB (one copy instead of two)

3. Remove jQuery (~87KB) — likely not needed in modern React app
   Action: audit jQuery usage, replace with native DOM APIs or React
   Savings: 87KB

4. Replace lodash with lodash-es + tree shaking (~60KB savings)
   lodash: 71KB → only what's imported: ~10KB
   Action: change to lodash-es, enable tree shaking

5. Route-based code splitting (no size savings but better initial load)
   Dashboard (180KB), Reports (90KB), Settings (40KB), Profile (35KB)
   → Only one page loads per navigation
   Initial bundle: app.js goes from 420KB to ~75KB (shared components)

Total estimated savings: 225 + 160 + 87 + 60 = 532KB reduction
Plus: route splitting means users download pages on-demand, not upfront
```

</details>

---

### Exercise 2 — Configure Vite for optimal production build

Write a `vite.config.ts` that:

- Splits React/React-DOM into their own chunk
- Splits any charting library into a charts chunk
- Enables Brotli compression
- Drops console.log in production
- Sets a chunk size warning at 300KB

<details>
<summary>Solution</summary>

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import compression from "vite-plugin-compression";

export default defineConfig(({ mode }) => ({
  plugins: [
    react(),
    // Brotli compression
    compression({
      algorithm: "brotliCompress",
      ext: ".br",
      deleteOriginFile: false,
    }),
    // Gzip fallback
    compression({
      algorithm: "gzip",
      ext: ".gz",
    }),
  ],

  build: {
    target: "es2020",
    minify: "terser",
    sourcemap: false,

    terserOptions: {
      compress: {
        drop_console: mode === "production",
        drop_debugger: true,
      },
    },

    chunkSizeWarningLimit: 300, // 300KB

    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes("node_modules")) {
            // React ecosystem: tiny, always needed, separate cache
            if (/(react|react-dom|react-router)/.test(id)) {
              return "react-vendor";
            }
            // Charting: large, only some pages need it
            if (/(recharts|chart\.js|d3|highcharts|apexcharts)/.test(id)) {
              return "charts-vendor";
            }
            // Everything else: general vendors
            return "vendors";
          }
        },
        chunkFileNames: "assets/[name]-[hash].js",
        assetFileNames: "assets/[name]-[hash].[ext]",
      },
    },
  },
}));
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/08-critical-rendering-path.md`](../browser-internals/08-critical-rendering-path.md) — How bundles affect CRP
- [`browser-internals/09-browser-caching.md`](../browser-internals/09-browser-caching.md) — Cache headers for bundles
- [`performance/07-memoization.md`](./07-memoization.md) — Runtime performance after loading
- [`performance/09-intersection-observer.md`](./09-intersection-observer.md) — Lazy loading with IntersectionObserver

---

<div align="center">

**Next:** [`performance/09-intersection-observer.md`](./09-intersection-observer.md) →

</div>
