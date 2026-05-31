# 11 — SVG Optimization

> **"SVG is the only graphics format that lives in the DOM. That's its superpower — CSS, JavaScript, accessibility, animations all work natively. It's also its limitation: every SVG element is a DOM node, and the DOM has costs."**

SVG (Scalable Vector Graphics) is the right choice for icons, illustrations, charts, and interactive diagrams. It scales perfectly to any resolution, integrates with CSS and JavaScript, and is accessible by default. But SVG performance degrades sharply with element count and animation complexity. This document covers SVG optimization from file-level to rendering-level: file size reduction, animation performance, DOM management, and when to switch from SVG to Canvas.

---

## 📚 Table of Contents

1. [SVG in the DOM — The Cost Model](#1-svg-in-the-dom--the-cost-model)
2. [SVG File Optimization](#2-svg-file-optimization)
3. [Inline SVG vs External SVG](#3-inline-svg-vs-external-svg)
4. [SVG Icon Systems](#4-svg-icon-systems)
5. [CSS Animations vs SMIL vs JavaScript](#5-css-animations-vs-smil-vs-javascript)
6. [Animating SVG with the Compositor](#6-animating-svg-with-the-compositor)
7. [SVG Filters — Performance Considerations](#7-svg-filters--performance-considerations)
8. [Responsive SVG](#8-responsive-svg)
9. [SVG for Data Visualization](#9-svg-for-data-visualization)
10. [SVG Sprites](#10-svg-sprites)
11. [Accessibility in SVG](#11-accessibility-in-svg)
12. [When to Switch from SVG to Canvas](#12-when-to-switch-from-svg-to-canvas)
13. [Build-Time SVG Optimization](#13-build-time-svg-optimization)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. SVG in the DOM — The Cost Model

Unlike Canvas (one bitmap element), SVG creates a DOM node for every shape, group, and text element. This has direct performance implications:

```
An SVG with 500 elements:
  500 DOM nodes
  500 × ~500 bytes = ~250KB DOM memory
  Each element: participates in layout, style matching, event bubbling
  Style recalculation: O(500 × CSS rules) on any class change
  Layout: O(500) for any geometry change
  Paint: O(visible elements) per frame

vs same image as Canvas:
  1 DOM node (the <canvas> element)
  ~2MB GPU texture (800×600 × 4 bytes)
  All 500 shapes: pixels in the bitmap, invisible to the DOM
```

### The SVG Sweet Spot

```
SVG performs well for:
  < 500 elements            → DOM cost manageable
  Static or rarely animated → no per-frame work
  Interactive (click/hover) → native event handling
  Accessibility required    → semantic elements, ARIA
  Zooming required          → infinite sharpness
  CSS theming               → fill="currentColor" etc.

SVG struggles with:
  > 1,000 elements          → DOM becomes heavy
  60fps animation of many elements → layout/paint per frame
  Complex filters (blur, drop-shadow) → expensive rasterization
  Real-time data (100+ updates/second) → DOM mutations expensive
```

---

## 2. SVG File Optimization

Exported SVG files from Illustrator, Figma, or Inkscape contain significant bloat.

### What SVGO Removes

```xml
<!-- Before optimization (Figma export): ~2.4KB -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "...">
<svg xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink"
     version="1.1"
     id="Layer_1"
     x="0px"
     y="0px"
     viewBox="0 0 24 24"
     style="enable-background:new 0 0 24 24;"
     xml:space="preserve">
  <g id="Group_123">
    <g id="icon-home">
      <path id="Path_456" class="st0" d="M12 2.69l5.5 5V20h-4v-6H10v6H6V7.69z"/>
      <!-- ... more paths with verbose attributes ... -->
    </g>
  </g>
  <style type="text/css">.st0{fill:#333333;}</style>
</svg>

<!-- After SVGO optimization: ~420 bytes (82% reduction) -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
  <path fill="#333" d="M12 2.69l5.5 5V20h-4v-6H10v6H6V7.69z"/>
</svg>
```

### SVGO Setup

```bash
# Install SVGO
npm install --save-dev svgo

# svgo.config.js
module.exports = {
  plugins: [
    { name: 'preset-default', params: {
        overrides: {
          // Keep viewBox (needed for responsive scaling)
          removeViewBox: false,
          // Keep IDs used by JavaScript
          cleanupIds: false,
        },
    }},
    // Collapse adjacent same-colored paths
    'collapseGroups',
    // Convert strokes to fills where possible
    'convertShapeToPath',
  ],
};

# Run on a file:
npx svgo icon.svg -o icon.min.svg

# Run on a directory:
npx svgo -r -f src/icons/ --output dist/icons/

# In package.json:
"optimize-svg": "svgo -r -f src/assets/icons/"
```

### Manual SVG Optimization Checklist

```
✅ Remove unused elements:
   <title>, <desc> (keep if needed for accessibility)
   <defs> with unreferenced definitions
   Hidden layers (display:none groups)

✅ Simplify paths:
   Reduce decimal precision: "M 10.5234567" → "M 10.52"
   Remove redundant commands: "L 10 20 L 30 20" → "H 30"
   Merge adjacent paths of same fill

✅ Remove attributes:
   id="" attributes not referenced by JS/CSS
   data-* attributes not needed in production
   Inline styles that duplicate presentation attributes

✅ Compress repeated structures with <use>:
   5 identical icons → 1 <symbol> + 5 <use>
```

---

## 3. Inline SVG vs External SVG

### Inline SVG `<svg>` in HTML

```html
<!-- Inline: full DOM access, CSS inheritance, no extra HTTP request -->
<button class="btn">
  <svg viewBox="0 0 24 24" width="20" height="20" aria-hidden="true">
    <path d="M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5" />
  </svg>
  Send
</button>
```

```css
/* CSS can reach inside inline SVG */
.btn:hover svg path {
  stroke: currentColor;
  fill: white;
}
```

**Best for:** Icons in UI components, SVGs needing CSS interaction.

### External SVG via `<img>`

```html
<!-- External: cached, no DOM access, no JavaScript targeting -->
<img src="/icons/logo.svg" alt="Company Logo" width="120" height="40" />
```

**Best for:** Logos, decorative illustrations, content images.

### External SVG via CSS `background-image`

```css
/* Inaccessible to DOM, but fine for decorative backgrounds */
.hero::before {
  content: "";
  background-image: url("/waves.svg");
  background-size: cover;
}
```

**Best for:** Decorative backgrounds, patterns.

### Comparison Table

| Method              | CSS Access  | JS Access |    ARIA    | HTTP Request | Cacheable |
| ------------------- | :---------: | :-------: | :--------: | :----------: | :-------: |
| Inline `<svg>`      |     ✅      |    ✅     |     ✅     |      ❌      |    ❌     |
| `<img src="*.svg">` |     ❌      |    ❌     | ✅ (`alt`) |      ✅      |    ✅     |
| CSS `background`    |     ❌      |    ❌     |     ❌     |      ✅      |    ✅     |
| `<object>`          | ✅ (shadow) |  Limited  |     ✅     |      ✅      |    ✅     |
| `<use>` sprite      |     ✅      |    ✅     |     ✅     |  ✅ (1 req)  |    ✅     |

---

## 4. SVG Icon Systems

### Method 1 — Hidden Sprite + `<use>` (Recommended)

Define all icons once in a hidden SVG sprite, reference with `<use>`:

```html
<!-- sprite.svg or inline at bottom of <body> -->
<svg style="display:none" aria-hidden="true">
  <defs>
    <symbol id="icon-home" viewBox="0 0 24 24">
      <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z" />
    </symbol>
    <symbol id="icon-search" viewBox="0 0 24 24">
      <path
        d="M15.5 14h-.79l-.28-.27A6.47 6.47 0 0 0 16 9.5 6.5 6.5 0 1 0 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5 4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14 9.5 11.99 14 9.5 14z"
      />
    </symbol>
    <!-- ... more icons ... -->
  </defs>
</svg>

<!-- Use anywhere on the page: -->
<button>
  <svg width="20" height="20" aria-hidden="true" focusable="false">
    <use href="#icon-home" />
  </svg>
  Home
</button>

<button>
  <svg width="20" height="20" aria-hidden="true" focusable="false">
    <use href="#icon-search" />
  </svg>
  Search
</button>
```

**Benefits:**

- Icons defined once, rendered via GPU compositing
- CSS `currentColor` works for theming
- One sprite file, cached by browser
- No per-icon HTTP requests

### Method 2 — Component-Based (React/Vue)

```typescript
// Icon component: inline SVG wrapped in a component
interface IconProps {
  name: keyof typeof icons;
  size?: number;
  className?: string;
  'aria-label'?: string;
}

const icons = {
  home:   <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>,
  search: <path d="M15.5 14h-.79l-.28-.27..."/>,
} as const;

export function Icon({ name, size = 20, className, 'aria-label': label }: IconProps) {
  return (
    <svg
      width={size}
      height={size}
      viewBox="0 0 24 24"
      className={className}
      aria-hidden={!label}
      aria-label={label}
      focusable="false"
    >
      {icons[name]}
    </svg>
  );
}
```

---

## 5. CSS Animations vs SMIL vs JavaScript

SVG can be animated three ways. They have very different performance profiles.

### SMIL Animations (Avoid)

```xml
<!-- SMIL: XML-based animation, deprecated in Chrome, inconsistent -->
<circle cx="50" cy="50" r="20">
  <animate attributeName="cx" from="50" to="200" dur="1s" repeatCount="indefinite"/>
</circle>
```

**Verdict:** Don't use SMIL. It's deprecated in Chrome and inconsistently supported.

### CSS Animations (Best for Simple)

```css
/* CSS animations on SVG: compositor-thread when using transform/opacity */
.icon-spin {
  animation: spin 1s linear infinite;
  transform-origin: center;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Color animations: paint-thread (fine for simple use) */
.icon-pulse path {
  animation: pulse 1.5s ease-in-out infinite;
}

@keyframes pulse {
  0%,
  100% {
    fill: #007bff;
  }
  50% {
    fill: #66b3ff;
  }
}
```

**Best for:** Loaders, spinners, icon hover effects, simple state transitions.

### JavaScript Animation (Best for Complex)

```javascript
// GSAP — most robust for complex SVG animations
import gsap from "gsap";

gsap.to("#icon-path", {
  x: 100,
  duration: 1,
  ease: "power2.out",
});

// SVG morphing (path shape animation)
gsap.to("#morphPath", {
  attr: { d: "M10 80 C 40 10, 65 10, 95 80 S 150 150, 180 80" },
  duration: 1,
});
```

```javascript
// Web Animations API — native, no library needed
const element = document.querySelector("#animated-path");
element.animate(
  [
    { transform: "translateX(0px)", opacity: 1 },
    { transform: "translateX(100px)", opacity: 0.5 },
  ],
  {
    duration: 1000,
    easing: "ease-out",
    fill: "forwards",
  },
);
```

---

## 6. Animating SVG with the Compositor

The same rule applies to SVG as to HTML: `transform` and `opacity` run on the compositor thread and don't trigger layout or paint.

```css
/* ✅ Compositor-only: no layout, no paint */
.animated-icon {
  transition:
    transform 0.3s ease,
    opacity 0.3s ease;
  will-change: transform; /* promote to GPU layer */
}

.animated-icon:hover {
  transform: scale(1.1);
  opacity: 0.9;
}

/* ❌ Paint-triggering: changes fill color every frame */
.animated-icon path {
  transition: fill 0.3s;
  fill: blue;
}
.animated-icon path:hover {
  fill: red; /* triggers paint per frame */
}
```

### SVG `transform-origin` Gotcha

```css
/* SVG elements have transform-origin at (0, 0) by default — top-left of SVG */
/* HTML elements have transform-origin at (50%, 50%) — center */

/* ❌ Rotates around SVG origin (0,0) — wrong for most icons */
.icon {
  transform: rotate(45deg);
}

/* ✅ Rotate around element's center */
.icon {
  transform: rotate(45deg);
  transform-origin: center;
}
/* Or: */
.icon {
  transform: rotate(45deg);
  transform-origin: 12px 12px;
} /* 50% of 24px icon */
```

---

## 7. SVG Filters — Performance Considerations

SVG filters (`<filter>`) enable effects impossible with CSS alone: Gaussian blur, color matrix, displacement map. They are also among the most expensive rendering operations.

```xml
<defs>
  <filter id="soft-glow">
    <feGaussianBlur stdDeviation="4" result="blurred"/>
    <feComposite in="SourceGraphic" in2="blurred" operator="over"/>
  </filter>
</defs>

<circle cx="50" cy="50" r="30" filter="url(#soft-glow)"/>
```

### Filter Performance Cost

```
feGaussianBlur (stdDeviation=4):
  For a 100×100 element: ~400 pixel operations × blur radius = expensive
  For a 1000×1000 element: 40,000 pixel operations × blur radius = very expensive

Animated filters: repaint on every frame → can easily drop to < 30fps
Static filters: rendered once, cached — acceptable
```

### Optimization: Limit Filter Region

```xml
<!-- Default: filter region is 10% larger than element on all sides -->
<!-- For large blur: this can be enormous -->
<filter id="blur" x="-5%" y="-5%" width="110%" height="110%">
  <feGaussianBlur stdDeviation="2"/>
</filter>

<!-- Explicit smaller region for tight filter effects -->
<filter id="shadow" x="-2%" y="-2%" width="104%" height="104%">
  <feDropShadow dx="0" dy="2" stdDeviation="1" flood-opacity="0.3"/>
</filter>
```

### Static Filter Optimization

```javascript
// Pre-render filtered element to image, use image thereafter
async function rasterizeSVGFilter(svgElement) {
  const xml = new XMLSerializer().serializeToString(svgElement);
  const img = new Image();

  await new Promise((resolve, reject) => {
    img.onload = resolve;
    img.onerror = reject;
    img.src = "data:image/svg+xml;charset=utf-8," + encodeURIComponent(xml);
  });

  const canvas = new OffscreenCanvas(img.width, img.height);
  canvas.getContext("2d").drawImage(img, 0, 0);
  return canvas.transferToImageBitmap();
}
// Use the ImageBitmap for display — filter computed once
```

---

## 8. Responsive SVG

```html
<!-- ✅ Responsive SVG: scales with container -->
<svg viewBox="0 0 200 100" xmlns="http://www.w3.org/2000/svg">
  <!-- viewBox defines the coordinate system -->
  <!-- No width/height attributes = fills container -->
</svg>

<style>
  .chart-container {
    width: 100%;
    max-width: 800px;
  }
  .chart-container svg {
    width: 100%;
    height: auto; /* maintains aspect ratio */
  }
</style>
```

### `preserveAspectRatio`

```xml
<!-- Default: "xMidYMid meet" — letterbox (like background-size: contain) -->
<svg viewBox="0 0 200 100" preserveAspectRatio="xMidYMid meet">

<!-- Crop (like background-size: cover) -->
<svg viewBox="0 0 200 100" preserveAspectRatio="xMidYMid slice">

<!-- Stretch (ignore aspect ratio) -->
<svg viewBox="0 0 200 100" preserveAspectRatio="none">

<!-- Align to top-left, no scaling -->
<svg viewBox="0 0 200 100" preserveAspectRatio="xMinYMin meet">
```

### Responsive Text in SVG

```javascript
// Text in SVG doesn't auto-wrap — handle responsive text with JavaScript
function fitTextToSVG(textEl, maxWidth) {
  textEl.setAttribute("textLength", String(maxWidth));
  textEl.setAttribute("lengthAdjust", "spacingAndGlyphs");
  // Or: use foreignObject for HTML text
}

// ✅ For wrapping text: use foreignObject
// <foreignObject x="10" y="10" width="200" height="100">
//   <p xmlns="http://www.w3.org/1999/xhtml">Text that wraps automatically</p>
// </foreignObject>
```

---

## 9. SVG for Data Visualization

SVG is widely used for charts. Performance depends on element count and update frequency.

### Efficient SVG Chart Updates

```javascript
// ❌ Rebuilding entire chart on data update
function updateChart(data) {
  svg.innerHTML = ""; // destroy all nodes
  data.forEach((d) => {
    const rect = document.createElementNS("http://www.w3.org/2000/svg", "rect");
    rect.setAttribute("x", String(xScale(d.x)));
    rect.setAttribute("height", String(yScale(d.y)));
    svg.appendChild(rect);
  });
}

// ✅ Update only changed elements (D3-style data join)
function updateChart(data) {
  const bars = svg.querySelectorAll(".bar");

  // Update existing bars
  data.forEach((d, i) => {
    if (i < bars.length) {
      // Update existing element
      bars[i].setAttribute("height", String(yScale(d.y)));
      bars[i].setAttribute("x", String(xScale(d.x)));
    } else {
      // Add new element
      const rect = document.createElementNS(
        "http://www.w3.org/2000/svg",
        "rect",
      );
      rect.classList.add("bar");
      rect.setAttribute("height", String(yScale(d.y)));
      rect.setAttribute("x", String(xScale(d.x)));
      svg.appendChild(rect);
    }
  });

  // Remove extra elements
  for (let i = data.length; i < bars.length; i++) {
    bars[i].remove();
  }
}
```

### CSS Transitions for Chart Animations

```css
/* ✅ CSS handles the animation — compositor thread for transform-based */
.bar {
  transition:
    height 0.5s ease-out,
    y 0.5s ease-out;
}

/* SVG height/y transitions go through layout → paint (not compositor)
   For compositor: use transform: scaleY() instead */
.bar-optimized {
  transform-origin: bottom;
  transition: transform 0.5s ease-out; /* compositor-only ✓ */
}
```

### Batch Attribute Updates

```javascript
// ❌ N setAttribute calls = N DOM mutations
function updateBars(bars, data) {
  bars.forEach((bar, i) => {
    bar.setAttribute("x", String(xScale(data[i].x)));
    bar.setAttribute("y", String(yScale(data[i].y)));
    bar.setAttribute("width", String(bandwidth));
    bar.setAttribute("height", String(yScale(0) - yScale(data[i].y)));
  });
}

// ✅ Batch via CSS transform (one mutation per element instead of four)
function updateBars(bars, data) {
  bars.forEach((bar, i) => {
    const scaleY = data[i].y / maxValue; // normalized
    bar.style.transform = `scaleY(${scaleY})`;
    // One CSS property change vs four attribute changes
  });
}
```

---

## 10. SVG Sprites

An SVG sprite bundles multiple icons into one file, reducing HTTP requests.

### Creating a Sprite

```xml
<!-- sprite.svg -->
<svg xmlns="http://www.w3.org/2000/svg" style="display:none">
  <symbol id="icon-home" viewBox="0 0 24 24">
    <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>
  </symbol>
  <symbol id="icon-user" viewBox="0 0 24 24">
    <path d="M12 12c2.7 0 4.8-2.1 4.8-4.8S14.7 2.4 12 2.4 7.2 4.5 7.2 7.2 9.3 12 12 12zm0 2.4c-3.2 0-9.6 1.6-9.6 4.8v2.4h19.2v-2.4c0-3.2-6.4-4.8-9.6-4.8z"/>
  </symbol>
</svg>
```

### Build-Time Sprite Generation

```javascript
// build-sprite.js — generates sprite from individual icon files
const fs = require("fs");
const path = require("path");

const iconsDir = "./src/icons";
const output = "./public/sprite.svg";

const icons = fs
  .readdirSync(iconsDir)
  .filter((f) => f.endsWith(".svg"))
  .map((file) => {
    const content = fs.readFileSync(path.join(iconsDir, file), "utf8");
    const id = path.basename(file, ".svg");
    // Extract inner SVG content, wrap in <symbol>
    const inner = content.match(/<svg[^>]*>([\s\S]*)<\/svg>/)?.[1] ?? "";
    const viewBox = content.match(/viewBox="([^"]+)"/)?.[1] ?? "0 0 24 24";
    return `<symbol id="icon-${id}" viewBox="${viewBox}">${inner}</symbol>`;
  })
  .join("\n");

fs.writeFileSync(
  output,
  `<svg xmlns="http://www.w3.org/2000/svg" style="display:none">\n${icons}\n</svg>`,
);
```

---

## 11. Accessibility in SVG

SVG is inherently accessible when authored correctly.

```xml
<!-- ✅ Standalone informative SVG: title + desc + role -->
<svg
  xmlns="http://www.w3.org/2000/svg"
  viewBox="0 0 24 24"
  role="img"
  aria-labelledby="title-id desc-id"
>
  <title id="title-id">Home</title>
  <desc id="desc-id">Navigate to the homepage</desc>
  <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>
</svg>

<!-- ✅ Decorative SVG: hidden from screen readers -->
<svg viewBox="0 0 24 24" aria-hidden="true" focusable="false">
  <path d="..."/>
</svg>

<!-- ✅ Icon inside button: button provides the accessible name -->
<button aria-label="Go to homepage">
  <svg viewBox="0 0 24 24" aria-hidden="true" focusable="false" width="20" height="20">
    <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>
  </svg>
</button>

<!-- ✅ Icon with text label: icon is decorative, text provides name -->
<button>
  <svg viewBox="0 0 24 24" aria-hidden="true" focusable="false" width="20" height="20">
    <path d="..."/>
  </svg>
  Home
</button>
```

---

## 12. When to Switch from SVG to Canvas

The threshold at which Canvas becomes preferable to SVG:

```
Element count:
  < 100 elements:   SVG — simpler, accessible, CSS-stylable
  100-500:          SVG — usually fine, profile first
  500-1,000:        Depends on animation — static SVG OK, animated → Canvas
  > 1,000:          Canvas or WebGL

Animation:
  Few slow transitions: SVG (CSS transitions)
  Many elements animating: Canvas (no DOM per element)
  60fps with > 100 animated elements: Canvas

Interactivity:
  Click/hover on shapes: SVG (native event handling)
  Pixel-level hit testing: Canvas (manual but flexible)
  Complex drag: Canvas (more control)

Text:
  Selectable/accessible text: SVG/HTML (don't use Canvas)
  Labels on a chart: SVG (if < 500 labels), Canvas (if > 500)

Data visualization:
  D3 with < 500 nodes: SVG
  D3 with > 1,000 nodes or real-time: Canvas
  Network graphs (> 500 nodes): Canvas or WebGL (Sigma.js, vis.js)
```

---

## 13. Build-Time SVG Optimization

### Vite Plugin for SVG

```typescript
// vite.config.ts — SVGR: transform SVG to React component + optimize
import svgr from "vite-plugin-svgr";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    svgr({
      svgrOptions: {
        plugins: ["@svgr/plugin-svgo", "@svgr/plugin-jsx"],
        svgoConfig: {
          plugins: [
            {
              name: "preset-default",
              params: { overrides: { removeViewBox: false } },
            },
          ],
        },
        // Add TypeScript types
        typescript: true,
        // Add title for accessibility
        titleProp: true,
      },
    }),
  ],
});

// Usage in component:
import { ReactComponent as HomeIcon } from "./icons/home.svg";
// Or with Vite SVGR:
import HomeIcon from "./icons/home.svg?react";
```

### Webpack with SVGO Loader

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.svg$/,
        use: [
          "@svgr/webpack",
          {
            loader: "svgo-loader",
            options: {
              plugins: [{ name: "preset-default" }],
            },
          },
        ],
      },
    ],
  },
};
```

---

## 14. Good Practices

### ✅ Always include `viewBox`

```xml
<!-- ✅ viewBox enables responsive scaling -->
<svg viewBox="0 0 24 24" width="24" height="24">
  <!-- width/height set default size; viewBox makes it scalable -->
</svg>
```

### ✅ Use `currentColor` for themeable icons

```xml
<!-- ✅ Icon inherits color from CSS -->
<svg viewBox="0 0 24 24" fill="currentColor">
  <path d="..."/>
</svg>

<style>
  .icon-primary { color: #007bff; }  /* icon inherits this */
  .icon-danger  { color: #dc3545; }
</style>
```

### ✅ Set `aria-hidden="true"` on decorative SVGs

```html
<!-- ✅ Screen readers skip decorative icons -->
<button>
  <svg aria-hidden="true" focusable="false">...</svg>
  Save
</button>
```

### ✅ Optimize before committing

```bash
# Run SVGO as part of the development workflow
npm run optimize-svg  # before committing new SVG files
```

### ✅ Use `<symbol>` + `<use>` for repeated icons

```html
<!-- ✅ Define once, use many times — no duplication in the DOM -->
<svg style="display:none">
  <symbol id="check-icon" viewBox="0 0 24 24">...</symbol>
</svg>

<!-- Many instances: lightweight <use> references, not full SVG copies -->
<svg><use href="#check-icon" /></svg>
<svg><use href="#check-icon" /></svg>
```

---

## 15. Bad Practices

### ❌ Animating `fill` or `stroke` color at high frequency

```css
/* ❌ Color animation triggers repaint every frame */
@keyframes colorPulse {
  0% {
    fill: red;
  }
  50% {
    fill: blue;
  }
  100% {
    fill: red;
  }
}
.pulse {
  animation: colorPulse 1s infinite;
}

/* ✅ Use opacity overlay for color "pulse" — compositor thread */
.pulse::after {
  background: blue;
  opacity: 0;
  animation: fade 1s infinite; /* opacity: compositor only */
}
```

### ❌ Embedding raster images in SVG

```xml
<!-- ❌ Base64 image in SVG: doubles the file size, prevents caching -->
<image href="data:image/png;base64,iVBORw0KGgoAAAANS..." width="100" height="100"/>

<!-- ✅ External reference: cacheable, separate HTTP cache entry -->
<image href="/images/photo.jpg" width="100" height="100"/>
```

### ❌ Complex SVG filters on animated elements

```xml
<!-- ❌ Gaussian blur on animating element: repaint every frame -->
<rect filter="url(#blur)" class="animated-rect"/>
<filter id="blur">
  <feGaussianBlur stdDeviation="5"/>
</filter>

<!-- ✅ Apply filter to a static background element instead,
       move the foreground element with compositor-friendly transform -->
```

---

## 16. Common Mistakes

### Mistake 1 — Missing `focusable="false"` on SVG in IE/Edge

```html
<!-- ❌ IE/Edge treats inline SVG as focusable by default -->
<button>
  <svg>...</svg>
  <!-- Tab key stops on SVG AND button separately -->
  Label
</button>

<!-- ✅ Explicitly set non-focusable -->
<button>
  <svg focusable="false" aria-hidden="true">...</svg>
  Label
</button>
```

### Mistake 2 — Scaling SVG with CSS `width`/`height` without `viewBox`

```xml
<!-- ❌ Without viewBox: setting CSS width scales the viewport, not the content -->
<svg width="100" height="100"> <!-- no viewBox -->
  <circle cx="50" cy="50" r="40"/>
</svg>

<!-- In CSS: width: 200px → SVG is 200×200px but circle is still at cx=50 cy=50 r=40 -->
<!-- The circle appears tiny in the top-left corner instead of centered -->

<!-- ✅ With viewBox: coordinates are relative to coordinate system -->
<svg viewBox="0 0 100 100"> <!-- with viewBox -->
  <circle cx="50" cy="50" r="40"/>
</svg>
<!-- Now CSS scaling works correctly -->
```

### Mistake 3 — Using SVG for large raster content

```xml
<!-- ❌ Large photographic image embedded in SVG -->
<svg>
  <image href="photo.jpg" width="1920" height="1080"/>
  <text>Caption</text>
</svg>

<!-- Use HTML <figure> + <img> + <figcaption> instead -->
<figure>
  <img src="photo.jpg" alt="...">
  <figcaption>Caption</figcaption>
</figure>
```

---

## 17. Interview-Level Explanation

> **"What are the performance considerations for SVG? How do you optimize SVG in production?"**

**Strong answer:**

> "SVG creates DOM nodes for every shape, group, and text element — it's not a single bitmap like Canvas. So the DOM cost model applies: each element participates in style matching, layout, and event propagation. For static SVGs with fewer than 500 elements, this is negligible. For animated SVGs or those with thousands of elements, it becomes a significant concern.
>
> The same CSS animation rules apply to SVG as to HTML: `transform` and `opacity` run on the compositor thread and don't trigger layout or paint. Animating `fill`, `stroke`, or `x`/`y` attributes triggers paint on every frame. So for icon animations — spinners, hover effects — I use CSS `transform` and `opacity` exclusively.
>
> For file optimization, SVGO removes ~60-80% of SVG file weight. Figma and Illustrator exports include DOCTYPE declarations, editor metadata, redundant IDs, verbose path data with excessive decimal precision. SVGO strips all of that. I run it as part of the build process on any SVG committed to the project.
>
> For icon systems, the `<symbol>` + `<use>` pattern is the right approach. Define all icons once in a hidden sprite SVG, reference them with `<use href="#icon-name"/>`. Icons become CSS-themeable via `fill="currentColor"`, there's one HTTP request for all icons, and there's no DOM duplication for repeated icons.
>
> The SVG vs Canvas decision depends on element count and animation frequency. Under 500 elements: SVG, because it gives you accessibility, CSS styling, and native events. Over 1,000 animated elements: Canvas, because the DOM cost becomes prohibitive. For real-time data visualization with many data points updating at high frequency, Canvas or WebGL is the correct choice."

---

## 18. Exercises

### Exercise 1 — SVG icon system implementation

Build a lightweight SVG icon system:

- Define 3 icons as symbols in a sprite
- Create a reusable `<icon>` web component that renders a `<use>` element
- Support `size` and `color` attributes
- Include proper accessibility attributes

<details>
<summary>Solution</summary>

```html
<!-- sprite.svg loaded once in document -->
<svg style="display:none" aria-hidden="true">
  <symbol id="icon-home" viewBox="0 0 24 24">
    <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z" />
  </symbol>
  <symbol id="icon-search" viewBox="0 0 24 24">
    <path
      d="M15.5 14h-.79l-.28-.27A6.47 6.47 0 0 0 16 9.5 6.5 6.5 0 1 0 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5 4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14 9.5 11.99 14 9.5 14z"
    />
  </symbol>
  <symbol id="icon-close" viewBox="0 0 24 24">
    <path
      d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"
    />
  </symbol>
</svg>

<script>
  class SvgIcon extends HTMLElement {
    static get observedAttributes() {
      return ["name", "size", "color", "label"];
    }

    connectedCallback() {
      this.#render();
    }
    attributeChangedCallback() {
      this.#render();
    }

    #render() {
      const name = this.getAttribute("name") ?? "home";
      const size = this.getAttribute("size") ?? "24";
      const color = this.getAttribute("color") ?? "currentColor";
      const label = this.getAttribute("label");

      this.innerHTML = `
      <svg
        width="${size}" height="${size}"
        fill="${color}"
        aria-hidden="${label ? "false" : "true"}"
        focusable="false"
        role="${label ? "img" : undefined}"
        aria-label="${label ?? ""}"
      >
        <use href="#icon-${name}"/>
      </svg>
    `;
    }
  }

  customElements.define("svg-icon", SvgIcon);
</script>

<!-- Usage -->
<button>
  <svg-icon name="home" size="20"></svg-icon>
  Home
</button>

<button aria-label="Search">
  <svg-icon name="search" size="20"></svg-icon>
</button>

<button>
  <svg-icon name="search" size="20" label="Search products"></svg-icon>
</button>
```

</details>

---

### Exercise 2 — Identify performance issues

```xml
<!-- Review this SVG and list every performance problem -->
<svg xmlns="http://www.w3.org/2000/svg"
     id="animated-logo"
     style="width: 300px; height: 300px;">
  <defs>
    <filter id="glow">
      <feGaussianBlur stdDeviation="8" result="blur"/>
      <feComposite in="SourceGraphic" in2="blur" operator="over"/>
    </filter>
  </defs>
  <g id="bg">
    <rect x="0" y="0" width="300" height="300" fill="#001f3f"/>
  </g>
  <g id="main" filter="url(#glow)">
    <circle class="orbit" cx="150" cy="150" r="80" fill="none" stroke="#4fc3f7" stroke-width="2"/>
    <circle class="planet" cx="230" cy="150" r="12" fill="#ff6b35"/>
  </g>
  <style>
    .orbit  { animation: spin 4s linear infinite; transform-origin: 150px 150px; }
    .planet { animation: spin 4s linear infinite; transform-origin: 150px 150px;
              fill: orange; transition: fill 0.3s; }
    .planet:hover { fill: red; }
    @keyframes spin { to { transform: rotate(360deg); } }
  </style>
</svg>
```

<details>
<summary>Answer</summary>

```
Performance issues:

1. feGaussianBlur filter on the animated group
   - stdDeviation=8 is aggressive
   - The group is animating (spinning) — repaint every frame with blur
   - Fix: apply blur to static background, not animated element
   - Or: reduce stdDeviation, apply filter sparingly

2. .planet { fill: orange } overrides the SVG attribute inline — creates specificity conflict
   CSS `fill` property vs SVG `fill` attribute — confusing and redundant

3. .planet transition: fill 0.3s — color transition triggers paint on every hover frame
   Fix: use opacity overlay for hover effect instead of changing fill

4. No viewBox — using fixed width/height means SVG won't scale responsively
   Fix: add viewBox="0 0 300 300", remove inline style width/height

5. SVG has no accessibility attributes — if it's decorative, add aria-hidden="true"
   If it has meaning, add role="img" and aria-label

6. `<style>` block inside SVG — should be external CSS for maintainability

7. .orbit and .planet have the same animation — could be simplified to one <g>
   rotating both together (one DOM element spinning instead of two)

Optimized approach:
  - One <g> with transform animation (compositor thread OK)
  - feGaussianBlur on a static background element only
  - viewBox for responsive scaling
  - aria-hidden="true" if decorative
  - Hover effect via opacity, not fill color change
```

</details>

---

## 🔗 Related Topics

- [`performance/10-canvas-optimization.md`](./10-canvas-optimization.md) — When to switch from SVG to Canvas
- [`browser-internals/05-paint-repaint.md`](../browser-internals/05-paint-repaint.md) — What SVG animations trigger
- [`browser-internals/06-composite-layers.md`](../browser-internals/06-composite-layers.md) — Compositor-only SVG animations
- [`performance/01-dom-optimization.md`](./01-dom-optimization.md) — SVG follows DOM cost model

---

<div align="center">

**Next:** [`performance/12-large-data-rendering.md`](./12-large-data-rendering.md) →

</div>
