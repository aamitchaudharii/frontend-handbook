# 03 — CSSOM

> **"The CSSOM is the invisible twin of the DOM. You can see the DOM in DevTools. You can modify it with JavaScript. But the CSSOM — the cascade, the computed values, the inheritance chain — operates entirely behind the scenes, and understanding it is what separates engineers who guess at CSS performance from those who know."**

The CSS Object Model (CSSOM) is one of the most misunderstood pieces of the browser. Most developers know CSS parsing blocks rendering. Few know why, or what the CSSOM actually contains, or how the browser resolves conflicting rules, or what `getComputedStyle` actually costs, or why certain CSS patterns trigger expensive recalculations. This document covers all of it.

---

## 📚 Table of Contents

1. [What the CSSOM Is](#1-what-the-cssom-is)
2. [How the CSSOM Is Built](#2-how-the-cssom-is-built)
3. [Why CSS Is Render-Blocking](#3-why-css-is-render-blocking)
4. [The Cascade — How Conflicts Are Resolved](#4-the-cascade--how-conflicts-are-resolved)
5. [Specificity — The Priority System](#5-specificity--the-priority-system)
6. [Inheritance](#6-inheritance)
7. [Computed vs Used vs Resolved Values](#7-computed-vs-used-vs-resolved-values)
8. [Style Recalculation — The Cost of CSS](#8-style-recalculation--the-cost-of-css)
9. [Selector Performance](#9-selector-performance)
10. [getComputedStyle — What It Actually Does](#10-getcomputedstyle--what-it-actually-does)
11. [CSS Custom Properties (Variables) Internals](#11-css-custom-properties-variables-internals)
12. [The CSSOM JavaScript API](#12-the-cssom-javascript-api)
13. [Critical CSS — Optimizing the Render-Blocking Path](#13-critical-css--optimizing-the-render-blocking-path)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. What the CSSOM Is

The CSSOM is a tree of objects that represents all CSS rules applied to a document — their structure, specificity, computed values, and inheritance relationships.

Unlike the DOM (which mirrors the HTML structure), the CSSOM has two distinct forms:

```
┌────────────────────────────────────────────────────────────────┐
│                         CSSOM                                   │
│                                                                  │
│  ┌──────────────────────────────────────────┐                  │
│  │  StyleSheetList (document.styleSheets)   │                  │
│  │  The "rule tree" — raw CSS rules as      │                  │
│  │  objects you can read and modify via JS  │                  │
│  │                                          │                  │
│  │  CSSStyleSheet → CSSRuleList             │                  │
│  │    └── CSSStyleRule { selectorText,      │                  │
│  │                        style: {...} }    │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                  │
│  ┌──────────────────────────────────────────┐                  │
│  │  Computed Style Tree                     │                  │
│  │  The resolved, computed values for       │                  │
│  │  every element — what the engine uses    │                  │
│  │  for layout and paint                    │                  │
│  │                                          │                  │
│  │  Element → { color: rgb(0,0,0),          │                  │
│  │              font-size: 16px,            │                  │
│  │              display: block, ... }       │                  │
│  └──────────────────────────────────────────┘                  │
└────────────────────────────────────────────────────────────────┘
```

The first form (the rule tree) is what you interact with via JavaScript's CSSOM API. The second (computed style tree) is what the browser uses internally and what `getComputedStyle()` exposes.

---

## 2. How the CSSOM Is Built

CSS is parsed in parallel with HTML parsing. When the HTML parser encounters a `<link rel="stylesheet">` or `<style>` tag, the browser:

1. **Dispatches a request** for external stylesheets (or reads inline styles)
2. **Tokenizes** the CSS text into tokens (selectors, property names, values, etc.)
3. **Parses** tokens into a rule tree
4. **Resolves** rules into computed styles for each element

### CSS Parsing Pipeline

```
CSS text (bytes)
      │
      ▼
┌─────────────────────────────────────────┐
│  TOKENIZER                               │
│  Converts bytes to CSS tokens:           │
│  ident, string, number, url, delim...    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  PARSER                                  │
│  Converts tokens to CSS rules:           │
│  QualifiedRule, AtRule, Declaration      │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  CSSOM RULE TREE                         │
│  StyleSheet → RuleList → Rules           │
│  (accessible via document.styleSheets)   │
└──────────────────┬──────────────────────┘
                   │  (combined with DOM)
                   ▼
┌─────────────────────────────────────────┐
│  COMPUTED STYLE RESOLUTION               │
│  For each element: cascade + inherit     │
│  → final computed value per property     │
└──────────────────┬──────────────────────┘
                   │
                   ▼
            Render Tree ready
```

### CSS Is Parsed Atomically

Unlike HTML (which parses incrementally), **CSS is parsed atomically** — the entire stylesheet must be received before the browser can finalize CSSOM construction. This is the fundamental reason CSS blocks rendering.

```
Why CSS can't be incremental:

At t=0, CSS says: .box { color: red; }
At t=100ms, more CSS arrives: .box { color: blue; }

If the browser had rendered after t=0:
  - User sees red box
  - Then blue box (flash of wrong color — FOUC)

Solution: wait for ALL CSS before rendering anything.
```

---

## 3. Why CSS Is Render-Blocking

CSS blocks rendering (but not parsing) for a fundamental reason: **a single CSS rule anywhere in the stylesheet can affect any element that already exists in the DOM**.

```css
/* This rule at line 10,000 of your stylesheet */
* {
  box-sizing: border-box;
}

/* Affects EVERY element in the entire document */
/* If browser had already rendered, it must repaint everything */
```

The browser has two options:

1. **Wait for all CSS** before rendering anything → render-blocking (what browsers do)
2. **Render as CSS loads** → constant flashes of unstyled content (FOUC)

Browsers choose option 1. This is the correct tradeoff: users prefer a slight delay over visual instability.

### The FOUC Problem

Before CSS loads completely:

```
Without render-blocking (hypothetical):

t=0ms:   HTML parsed → page renders with browser default styles
         (black text, blue links, no layout)

t=200ms: CSS loads → page re-renders with correct styles
         Flash! User sees unstyled content then styled content

With render-blocking (actual browser behavior):

t=0ms:   HTML parsed (not rendered yet — waiting for CSS)
t=200ms: CSS loaded → CSSOM built → render tree built → render
         First render is the correctly styled page. No flash.
```

### Which Resources Are Render-Blocking

```html
<!-- ✅ Render-blocking (expected and necessary) -->
<link rel="stylesheet" href="styles.css" />

<!-- ✅ Render-blocking only for matching media -->
<link rel="stylesheet" href="print.css" media="print" />
<!-- Not render-blocking on screen! Only blocks print rendering -->

<link rel="stylesheet" href="mobile.css" media="(max-width: 768px)" />
<!-- Only render-blocking when viewport matches the media query -->

<!-- ❌ Render-blocking unnecessarily -->
<link rel="stylesheet" href="everything.css" />
<!-- Including print styles in the main stylesheet makes it render-blocking
     on all devices — even print styles users will never see -->
```

---

## 4. The Cascade — How Conflicts Are Resolved

The "C" in CSS stands for Cascading. When multiple rules target the same element and property, the cascade determines which one wins.

### The Cascade Algorithm

The browser resolves conflicts using this priority order (highest to lowest):

```
1. !important declarations (from any origin)
   ├── User-agent !important
   ├── User !important
   └── Author !important  ← your CSS with !important

2. Normal declarations
   ├── Inline styles (style="...")
   ├── Author styles (your stylesheets)
   ├── User styles (user's browser preferences)
   └── User-agent styles (browser defaults)

3. Within same origin: Specificity
   (higher specificity wins)

4. Within same specificity: Source Order
   (last declaration wins)
```

### Cascade Origins

```
User-agent (browser defaults):
  div { display: block; }
  h1 { font-size: 2em; font-weight: bold; }
  a { color: blue; text-decoration: underline; }

User (browser settings / accessibility):
  /* User may set minimum font size, color preferences */

Author (your CSS — highest priority without !important):
  h1 { color: navy; }

Inline (style attribute — highest author priority):
  <h1 style="color: red;">  /* wins over stylesheet h1 { color: navy } */
```

### The `!important` Nuclear Option

```css
/* Author !important beats inline styles */
h1 {
  color: red !important; /* beats <h1 style="color: blue"> */
}

/* User-agent !important beats author !important */
/* (rare, used for accessibility — can't be overridden) */
```

**Production rule:** `!important` is a sign of a specificity problem. Use it sparingly — it breaks the natural cascade and makes future style changes unpredictable.

---

## 5. Specificity — The Priority System

Within the same cascade origin and layer, **specificity** determines which rule wins. It's calculated as a three-part number: **(A, B, C)**.

### Specificity Calculation

```
A = Number of ID selectors (#id)
B = Number of class selectors (.class), attribute selectors ([attr]),
    and pseudo-classes (:hover, :focus, :nth-child...)
C = Number of type selectors (div, p, h1...) and pseudo-elements
    (::before, ::after...)

Universal selector (*), combinators (+, >, ~, space): 0 specificity
```

### Specificity Examples

```css
*                         { } /* (0,0,0) — no specificity */
li                        { } /* (0,0,1) */
li::before                { } /* (0,0,2) — type + pseudo-element */
.container                { } /* (0,1,0) */
li.active                 { } /* (0,1,1) — type + class */
#header                   { } /* (1,0,0) */
#header .nav li:hover     { } /* (1,1,2) — id + class + type + pseudo-class */
style=""                    /* (1,0,0,0) — inline style beats all */
```

### The Specificity Comparison

```css
/* Which color wins for .btn#submit? */
.btn {
  color: blue;
} /* (0,1,0) */
#submit {
  color: green;
} /* (1,0,0) */

/* (1,0,0) > (0,1,0): green wins */
/* Even though .btn is later in the source */
```

### Specificity Traps

```css
/* ❌ TRAP: Increasing specificity to override styles creates an arms race */
.nav li a {
  color: blue;
} /* (0,1,3) */
.nav li a.active {
  color: red;
} /* (0,2,3) */
.sidebar .nav li a.active {
  color: green;
} /* (0,3,3) */
/* Cascade is now unpredictable — each rule needs higher specificity */

/* ✅ BETTER: Keep specificity flat */
.nav-link {
  color: blue;
} /* (0,1,0) */
.nav-link--active {
  color: red;
} /* (0,1,0) — same specificity, source order wins */
```

---

## 6. Inheritance

Some CSS properties are **inherited** by default — child elements receive the computed value of the property from their parent. Others are **non-inherited** — each element starts fresh.

### Inherited vs Non-Inherited Properties

```css
/* INHERITED by default (value flows down to children): */
color          /* text color */
font-family
font-size
font-weight
line-height
text-align
visibility
cursor
/* ...and most text-related properties */

/* NOT inherited (each element gets initial/default value): */
background
border
margin
padding
width
height
display
position
overflow
/* ...and most box-model properties */
```

### How Inheritance Works

```html
<div style="color: navy; font-size: 18px;">
  <p>
    <!-- inherits color: navy, font-size: 18px -->
    <strong>
      <!-- inherits color: navy, font-size: 18px -->
      Bold text
      <!-- final computed color: navy, font-size: 18px -->
    </strong>
  </p>
</div>
```

### Explicit Inheritance Controls

```css
.element {
  color: inherit; /* explicitly inherit from parent (even for non-inherited) */
  color: initial; /* reset to CSS specification default */
  color: unset; /* inherited properties → inherit; non-inherited → initial */
  color: revert; /* reset to browser stylesheet (user-agent) default */
}
```

---

## 7. Computed vs Used vs Resolved Values

These three value types are often confused but represent different stages of CSS processing:

### Specified Value

What you wrote in the stylesheet:

```css
.element {
  font-size: 1.5em; /* specified: 1.5em (relative) */
  width: 50%; /* specified: 50% (relative) */
  color: red; /* specified: red (keyword) */
}
```

### Computed Value

The specified value after **resolving relative values where possible**, applying inheritance, and normalizing keywords. Computed at **style recalculation** time. Does not require layout knowledge.

```css
/* Specified → Computed */
font-size: 1.5em    → 24px   (if parent is 16px: 1.5 × 16 = 24)
color: red          → rgb(255, 0, 0)  (normalized to rgb())
visibility: hidden  → hidden (keywords kept as-is)
width: 50%          → 50%    (can't resolve % without layout — stays as %)
```

### Used Value

The computed value after **layout** — all relative lengths resolved to absolute pixels:

```css
/* Computed → Used */
width: 50%   → 400px  (if parent is 800px wide)
height: auto → 120px  (computed after content flows)
```

### Resolved Value

What `getComputedStyle()` returns — **either computed or used**, depending on the property:

```javascript
const style = window.getComputedStyle(element);

// For width: returns USED value (pixels, after layout)
style.width; // "400px" (not "50%")

// For font-size: returns COMPUTED value
style.fontSize; // "24px"

// For color: returns COMPUTED value
style.color; // "rgb(255, 0, 0)"
```

### Why the Distinction Matters

```javascript
// ❌ Misconception: getComputedStyle returns what you wrote in CSS
element.style.width = "50%";
getComputedStyle(element).width; // "400px" — NOT "50%"
// This is the USED value (resolved after layout), not the specified value

// ✅ To get the specified value: use element.style
element.style.width; // "50%" — what was explicitly set via JS

// element.style only has values set directly via JS or inline style=""
// It's EMPTY for stylesheet-defined styles
element.style.color; // "" — empty if color was set in a stylesheet
```

---

## 8. Style Recalculation — The Cost of CSS

**Style recalculation** is the browser process of re-computing the computed styles for elements after something changes. It's the first step that can be triggered by DOM mutations or class changes.

### What Triggers Style Recalculation

```javascript
// ANY of these trigger style recalculation:
element.classList.add("active");
element.classList.remove("visible");
element.setAttribute("data-state", "open");
element.style.color = "red";
element.id = "newId";

document.body.appendChild(newElement); // new element needs styles computed
document.body.removeChild(element); // sibling/parent styles may change

// CSS class changes trigger matching all rules against all elements
// (in all affected subtrees)
```

### The Recalculation Process

```
1. Invalidation: mark which elements have potentially changed styles
   - Scope depends on the selector that matched the changed element
   - Universal selector (*) invalidates EVERYTHING
   - ID selector (#id) invalidates only that element

2. Matching: for each invalidated element, find which CSS rules match it
   - Walk all CSS rules, check selectors against the element

3. Cascade: apply the cascade algorithm to resolve conflicts
   - Compare specificity, origin, source order

4. Inheritance: propagate inherited values down the tree

5. Compute: resolve relative values, normalize keywords
```

### Recalculation Cost at Scale

```javascript
// Adding one class to body can trigger recalculation of
// EVERY element in the document (if CSS uses body-level selectors)

// Example expensive pattern:
body.dark-mode * { background: #1a1a1a; } /* universal selector */
// Changing body's class → recalculate styles for every single element

// ✅ Better: scope to specific components
[data-theme="dark"] .card { background: #1a1a1a; }
[data-theme="dark"] .nav  { background: #2a2a2a; }
// Only recalculates elements matching these specific selectors
```

### Forcing Style Recalculation from JavaScript

```javascript
// These force style recalculation (similar to layout-forcing reads):
window.getComputedStyle(element);
element.currentStyle; // IE only

// After a class change, if you immediately read computed styles:
element.classList.add("highlight");
window.getComputedStyle(element).color; // forces immediate recalculation
// (not a layout, but still a style calculation — can be expensive)
```

---

## 9. Selector Performance

Selectors are evaluated **right-to-left** by the browser's CSS engine. Understanding this explains why some selectors are more expensive than others.

### Right-to-Left Evaluation

```css
/* This selector: */
.nav > ul > li > a:hover

/* Is evaluated right-to-left:
   1. Find all <a> elements with :hover state
   2. Check if parent is <li>
   3. Check if that <li>'s parent is <ul>
   4. Check if that <ul>'s parent has class .nav
*/
```

Why right-to-left? Because the **key selector** (rightmost part) is used to find initial candidates. Browsers index elements by their key selector (tag name, class, ID) for fast lookup. Starting from the right and filtering leftward is more efficient than starting from the left.

### Selector Cost (Rough Ordering)

```
FASTEST → SLOWEST

#id                         — hash table lookup, O(1)
.class                      — hash set lookup, very fast
element                     — tag name lookup, fast
[attribute]                 — attribute index lookup, moderate
[attribute^=value]          — partial match, slower
:pseudo-class               — computed state check, varies
*                           — ALL elements must be checked
.ancestor .descendant       — expensive: for every .descendant,
                               walk up the entire DOM tree checking for .ancestor
```

### The Universal Selector Trap

```css
/* ❌ Forces the browser to check EVERY element */
* {
  box-sizing: border-box;
}
/* Better: only on elements that need it */

/* ❌ Very expensive: for every element, walk up looking for .container */
.container * {
  color: navy;
}

/* ❌ Descendant selector from a common ancestor */
.page div {
  margin: 0;
}
/* For every <div> in the document, walk up checking for .page */
```

### Practical Selector Advice

In 2024, selector performance is rarely the bottleneck — browsers have highly optimized CSS engines. Focus on:

1. **Avoiding `*` in hot paths** (animations, frequently added/removed classes)
2. **Keeping selectors flat** (BEM-style: `.card__title` vs `.card > header > h2`)
3. **Avoiding deeply nested descendant selectors** (`.a .b .c .d .e`)

```css
/* ✅ Flat BEM selectors — easy to match, explicit */
.card {
}
.card__title {
}
.card__body {
}
.card--highlighted {
}

/* ❌ Deeply nested — harder to match, tightly coupled to HTML structure */
.main-content > .article-list > article > .article-header > h2 {
}
```

---

## 10. getComputedStyle — What It Actually Does

`window.getComputedStyle(element)` is the JavaScript gateway to the browser's computed style tree. It's more expensive than it appears.

### What It Does

```javascript
const styles = window.getComputedStyle(element);
// Returns a live CSSStyleDeclaration reflecting computed styles
// "Live" = updates when styles change

styles.color; // "rgb(0, 0, 0)"
styles.fontSize; // "16px"
styles.display; // "block"
styles.width; // "400px" (used value — includes layout)
styles.getPropertyValue("background-color"); // "rgba(0, 0, 0, 0)"
```

### The Hidden Cost

`getComputedStyle` can **force style recalculation** if styles are dirty:

```javascript
element.classList.add("active"); // invalidates styles
// At this point: styles are "dirty" — not yet recalculated

const color = getComputedStyle(element).color;
// Browser MUST recalculate styles NOW to return accurate value
// This is a forced synchronous style recalculation
```

Additionally, `width`, `height`, and other geometric properties from `getComputedStyle` require **layout** to be current:

```javascript
element.style.width = "50%";
getComputedStyle(element).width; // "400px" — forces layout to resolve %
// This is equivalent to reading offsetWidth — triggers layout
```

### Efficient Use of getComputedStyle

```javascript
// ❌ Reading in a loop — forces recalculation per item
elements.forEach((el) => {
  const color = getComputedStyle(el).color; // potentially N recalculations
  processColor(color);
});

// ✅ Read once if all elements share the style
const color = getComputedStyle(elements[0]).color;
elements.forEach((el) => processColor(color));

// ✅ Use CSS Custom Properties for dynamic values — cheaper to read
// CSS: --brand-color: #007bff;
const brandColor = getComputedStyle(document.documentElement)
  .getPropertyValue("--brand-color")
  .trim();
// This doesn't force layout — custom properties don't require geometric resolution
```

---

## 11. CSS Custom Properties (Variables) Internals

CSS Custom Properties (variables) are stored in the CSSOM differently from regular properties — with implications for performance.

### How They Work Internally

```css
:root {
  --primary: #007bff;
  --spacing: 8px;
}

.button {
  color: var(--primary);
  padding: var(--spacing);
}
```

Custom properties are **inherited** like regular CSS inheritance — they cascade down the DOM tree. When you change a custom property on an ancestor, all descendants that use it need style recalculation.

```javascript
// Changing a custom property on :root invalidates ALL elements that use it
document.documentElement.style.setProperty("--primary", "#ff0000");
// Every element using var(--primary) needs style recalculation
// This can be expensive for large trees
```

### CSS Variables vs JavaScript Updates

```javascript
// ❌ Updating many individual properties — N style mutations
element.style.color = "#ff0000";
element.style.background = "#001f3f";
element.style.borderColor = "#ff0000";
// Three separate style invalidations

// ✅ Update one variable, CSS handles the rest
// CSS: .button { color: var(--accent); background: var(--bg); border-color: var(--accent); }
document.documentElement.style.setProperty("--accent", "#ff0000");
document.documentElement.style.setProperty("--bg", "#001f3f");
// Two variable changes → CSS engine resolves all dependent properties
```

### Custom Properties in Animations

Custom properties can be animated with `@property` (Houdini):

```css
@property --angle {
  syntax: "<angle>";
  initial-value: 0deg;
  inherits: false;
}

.spinner {
  background: conic-gradient(from var(--angle), navy, skyblue);
  animation: spin 2s linear infinite;
}

@keyframes spin {
  to {
    --angle: 360deg;
  }
}
/* CSS engine animates --angle natively — no JavaScript needed */
```

---

## 12. The CSSOM JavaScript API

The browser exposes the CSSOM via JavaScript for programmatic reading and modification of stylesheets.

### Accessing Stylesheets

```javascript
// All stylesheets in the document
const sheets = document.styleSheets; // StyleSheetList
const firstSheet = sheets[0]; // CSSStyleSheet

// Sheet properties
firstSheet.href; // URL of external stylesheet (null for inline)
firstSheet.disabled; // false (true to disable the sheet)
firstSheet.media; // MediaList

// Accessing rules
const rules = firstSheet.cssRules; // CSSRuleList (read-only collection)
```

### Reading Rules

```javascript
const sheet = document.styleSheets[0];

for (const rule of sheet.cssRules) {
  if (rule instanceof CSSStyleRule) {
    console.log("Selector:", rule.selectorText);
    console.log("Color:", rule.style.color);
    console.log("Font-size:", rule.style.fontSize);
  }

  if (rule instanceof CSSMediaRule) {
    console.log("Media:", rule.conditionText);
    for (const innerRule of rule.cssRules) {
      // rules inside @media block
    }
  }
}
```

### Modifying Rules Dynamically

```javascript
// Add a rule to a stylesheet
const sheet = document.styleSheets[0];
sheet.insertRule(".highlight { background: yellow; }", sheet.cssRules.length);

// Delete a rule
sheet.deleteRule(0); // removes first rule

// Modify a specific rule's property
sheet.cssRules[0].style.color = "red";

// Create a new stylesheet programmatically
const newSheet = document.createElement("style");
document.head.appendChild(newSheet);
newSheet.sheet.insertRule(".dynamic { color: purple; }", 0);
```

### CSS Typed OM (Houdini — Modern)

```javascript
// CSS Typed OM: work with CSS values as typed objects, not strings
element.attributeStyleMap.set("opacity", CSS.number(0.5));
element.attributeStyleMap.set(
  "transform",
  new CSSTranslate(CSS.px(100), CSS.px(0)),
);

// Reading typed values
const opacity = element.computedStyleMap().get("opacity");
// → { value: 0.5, type: 'number' } — not the string "0.5"

// Typed OM is faster than string parsing for frequent updates
// (animation engines, games, data visualizations)
```

---

## 13. Critical CSS — Optimizing the Render-Blocking Path

Since all CSS is render-blocking, the strategy is to deliver only what's needed for the initial render as fast as possible.

### The Critical CSS Technique

```html
<!-- ✅ Inline only above-the-fold CSS → no blocking network request -->
<head>
  <style>
    /* Critical CSS: only what's needed for first viewport */
    *,
    *::before,
    *::after {
      box-sizing: border-box;
    }
    body {
      margin: 0;
      font-family: -apple-system, sans-serif;
    }
    .hero {
      background: navy;
      color: white;
      padding: 4rem 2rem;
    }
    .hero__title {
      font-size: 2.5rem;
      margin: 0;
    }
    .nav {
      display: flex;
      padding: 1rem;
      background: white;
    }
  </style>

  <!-- Full CSS loads asynchronously — does not block initial render -->
  <link
    rel="preload"
    href="/styles.css"
    as="style"
    onload="this.onload=null;this.rel='stylesheet'"
  />
  <noscript><link rel="stylesheet" href="/styles.css" /></noscript>
</head>
```

### Media Query Splitting

```html
<!-- ✅ Non-matching media queries do NOT block rendering -->
<link rel="stylesheet" href="base.css" />
<!-- blocks all -->
<link rel="stylesheet" href="print.css" media="print" />
<!-- only blocks print -->
<link rel="stylesheet" href="mobile.css" media="(max-width: 768px)" />
<!-- On a desktop viewport: this doesn't block rendering! -->
<!-- On a mobile viewport: this blocks rendering. -->
<!-- But it still downloads on all devices — just doesn't block desktop render. -->
```

### Preconnect for External CSS

```html
<!-- Start TCP/TLS handshake early for external CSS providers -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Then the actual CSS request is faster -->
<link
  rel="stylesheet"
  href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600"
/>
```

### Font Loading Optimization

Fonts are a common render-blocking culprit:

```css
/* ✅ font-display: swap — show fallback font immediately, swap when loaded */
/* No invisible text during font load */
@font-face {
  font-family: "Inter";
  src: url("/fonts/inter.woff2") format("woff2");
  font-display: swap; /* show fallback immediately, no FOIT */
}
```

```html
<!-- Preload critical fonts to reduce FOUT -->
<link
  rel="preload"
  href="/fonts/inter-400.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

---

## 14. Good Practices

### ✅ Keep selectors flat and specific

```css
/* ✅ BEM-style: flat, predictable specificity */
.card {
}
.card__header {
}
.card__title {
}
.card--featured {
}

/* ❌ Deeply nested: high specificity, fragile */
.page .content .cards .card .header h2 {
}
```

### ✅ Use media queries to avoid unnecessary render-blocking

```html
<!-- ✅ Only block rendering with CSS that applies to the current context -->
<link rel="stylesheet" href="print.css" media="print" />
<link rel="stylesheet" href="wide.css" media="(min-width: 1200px)" />
```

### ✅ Batch class changes to minimize style recalculations

```javascript
// ❌ Three separate style recalculations
element.classList.add("visible");
element.classList.add("active");
element.classList.add("highlighted");

// ✅ One style recalculation (classList.add accepts multiple values)
element.classList.add("visible", "active", "highlighted");

// Or toggle with one class that represents the combined state
element.classList.add("card--active-highlighted");
```

### ✅ Avoid `!important` — fix the specificity problem instead

```css
/* ❌ Using !important to override a selector you should just fix */
.button {
  color: red !important;
}

/* ✅ Fix the selector specificity or structure */
.nav .button {
  color: blue;
} /* original — too specific */
.button--nav {
  color: blue;
} /* better — BEM modifier, can be overridden */
```

### ✅ Use `content-visibility: auto` for off-screen content

```css
/* ✅ Skip rendering of off-screen sections entirely */
.article-section {
  content-visibility: auto;
  contain-intrinsic-size: 0 500px; /* estimated height hint */
}
/* Browser skips style recalculation, layout, and paint for off-screen sections */
/* Large performance win for long pages */
```

---

## 15. Bad Practices

### ❌ Using `*` selector in hot paths

```css
/* ❌ Universal selector — matches every element */
* {
  transition: all 0.3s ease;
}
/* Forces the browser to animate EVERY CSS property of EVERY element */
/* Extremely expensive — don't do this */

/* ✅ Be explicit about what you're transitioning */
.interactive {
  transition:
    transform 0.3s ease,
    opacity 0.3s ease;
}
```

### ❌ Deep descendant selectors

```css
/* ❌ For every <span> on the page, walk up checking for .container, .wrapper, .content */
.container .wrapper .content p span {
  color: navy;
}

/* ✅ Target directly */
.content-text {
  color: navy;
}
```

### ❌ Using `getComputedStyle` in loops for layout properties

```javascript
// ❌ Forces style recalculation AND layout per iteration
elements.forEach((el) => {
  const width = getComputedStyle(el).width; // "400px" — triggers layout
  adjustWidth(el, parseFloat(width));
});

// ✅ Use ResizeObserver or batch the reads
const ro = new ResizeObserver((entries) => {
  entries.forEach(({ target, contentRect }) => {
    adjustWidth(target, contentRect.width); // width provided, no forced layout
  });
});
elements.forEach((el) => ro.observe(el));
```

### ❌ Animating properties that trigger layout or paint

```css
/* ❌ Triggers full layout on every animation frame */
@keyframes expand {
  from {
    width: 100px;
    height: 50px;
  }
  to {
    width: 200px;
    height: 100px;
  }
}

/* ✅ Use transform — composite only */
@keyframes expand {
  from {
    transform: scale(1);
  }
  to {
    transform: scale(2);
  }
}
```

---

## 16. Common Mistakes

### Mistake 1 — Thinking `element.style` reflects stylesheet styles

```javascript
// element.style ONLY reflects INLINE styles (set via JS or style="...")
// It is EMPTY for styles set in stylesheets

div.style.color; // "" — even if color: red is in a stylesheet

// ✅ Use getComputedStyle for stylesheet-applied values
getComputedStyle(div).color; // "rgb(255, 0, 0)"
```

### Mistake 2 — Expecting immediate style changes

```javascript
element.style.display = "none";
// Display hasn't changed yet in the rendered page!
// The change is queued — browser updates styles before next paint

// Reading a style property after setting another forces recalculation:
element.style.display = "block";
getComputedStyle(element).height; // forces immediate recalculation + possible layout
```

### Mistake 3 — Specificity wars leading to `!important` abuse

```css
/* Theme needs to override component: */
.primary-btn {
  color: white !important;
} /* ← had to add !important */
/* Now to override the override: */
.disabled .primary-btn {
  color: gray !important;
} /* !important escalation */
/* Now even more !important needed... */
```

The fix is always to resolve the underlying specificity conflict, not to add `!important`.

### Mistake 4 — Using `transition: all`

```css
/* ❌ Transitions EVERY property — including expensive ones you don't want */
.card {
  transition: all 0.3s ease;
}
.card:hover {
  /* Even a non-visual property change triggers transition */
}

/* ✅ Be explicit */
.card {
  transition:
    transform 0.3s ease,
    box-shadow 0.3s ease;
}
```

---

## 17. Interview-Level Explanation

> **"What is the CSSOM? Why does CSS block rendering? How does the cascade work?"**

**Strong answer:**

> "The CSSOM is the browser's in-memory representation of all CSS rules. It has two parts: the rule tree (what `document.styleSheets` exposes — the raw rules as objects), and the computed style tree — the resolved, computed styles for every element that the browser uses for layout and paint.
>
> CSS blocks rendering because of the cascade. A single CSS rule anywhere in a stylesheet can affect any element already in the DOM. If the browser rendered content before all CSS was loaded, you'd get a flash of unstyled content — text would appear in browser-default styles, then jump to the correct styles. So the browser waits for all CSS to build the complete CSSOM before rendering anything. HTML parsing continues during CSS loading — it's rendering that waits.
>
> The cascade resolves conflicts between rules using this priority order: first, `!important` declarations; then inline styles; then author stylesheets; then user preferences; then browser defaults. Within the same origin and layer, specificity decides: IDs outrank classes which outrank element selectors, calculated as a three-tuple (A, B, C). Same specificity? The last declaration in source order wins.
>
> Style recalculation is the process of recomputing computed styles after something changes — like adding a class or modifying a property. It can be triggered from JavaScript via class changes, attribute changes, or DOM mutations. `getComputedStyle()` can force an immediate synchronous recalculation if styles are dirty, which is why it should be avoided in tight loops.
>
> For performance, the key optimizations are: using `transform` and `opacity` for animations (they skip layout and paint entirely), keeping selectors flat to make matching cheaper, using CSS containment to limit recalculation scope, and inlining critical CSS to eliminate the render-blocking network request."

---

## 18. Exercises

### Exercise 1 — Specificity calculation

Calculate the specificity of each selector and predict which wins:

```css
/* A */
div.container p {
  color: red;
}
/* B */
.container > p {
  color: blue;
}
/* C */
p {
  color: green;
}
/* D */
#main p {
  color: orange;
}
/* E */
.container p.highlight {
  color: purple;
}
```

Which color does `<p class="highlight">` inside `<div id="main" class="container">` get?

<details>
<summary>Answer</summary>

```
A: div.container p   → (0, 1, 2) — 1 class + 2 type selectors
B: .container > p    → (0, 1, 1) — 1 class + 1 type selector
C: p                 → (0, 0, 1) — 1 type selector
D: #main p           → (1, 0, 1) — 1 ID + 1 type selector
E: .container p.highlight → (0, 2, 1) — 2 classes + 1 type selector

Comparison for a <p class="highlight"> inside <div id="main" class="container">:
  D wins: (1, 0, 1) — ID selector always beats class/element selectors

Color: orange (#main p matches)

Note: If there were also an inline style, that would beat D.
If D didn't match, E (0,2,1) would beat A (0,1,2).
```

</details>

---

### Exercise 2 — Trace the computed value

```css
:root {
  font-size: 16px;
}
.parent {
  font-size: 1.5em;
}
.child {
  font-size: 0.75em;
}
.grandchild {
  font-size: 1em;
}
```

```html
<div class="parent">
  <div class="child">
    <div class="grandchild">What is my font-size in pixels?</div>
  </div>
</div>
```

<details>
<summary>Answer</summary>

```
:root: 16px
.parent: 1.5em × 16px = 24px
.child: 0.75em × 24px = 18px
.grandchild: 1em × 18px = 18px

Answer: 18px

Key: em resolves relative to the PARENT's font-size (computed value),
not the root font-size. Each computation uses the already-computed
parent value. This is why em-based sizing can compound unexpectedly
in deeply nested structures.

(rem always resolves relative to :root — much more predictable)
```

</details>

---

### Exercise 3 — Fix the blocking CSS

This page has poor render-blocking performance. Identify all render-blocking issues and propose fixes.

```html
<!DOCTYPE html>
<html>
  <head>
    <link rel="stylesheet" href="reset.css" />
    <link rel="stylesheet" href="typography.css" />
    <link rel="stylesheet" href="layout.css" />
    <link rel="stylesheet" href="components.css" />
    <link rel="stylesheet" href="print.css" />
    <link rel="stylesheet" href="animations.css" />
    <link
      rel="stylesheet"
      href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700"
    />
  </head>
  <body>
    ...
  </body>
</html>
```

<details>
<summary>Solution</summary>

```html
<!DOCTYPE html>
<html>
  <head>
    <!-- 1. Preconnect to font provider — reduces DNS + TLS overhead -->
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

    <!-- 2. Inline critical CSS (above-the-fold styles only) -->
    <style>
      /* Reset essentials + above-fold layout */
      *,
      *::before,
      *::after {
        box-sizing: border-box;
        margin: 0;
      }
      body {
        font-family: -apple-system, sans-serif;
      }
      /* First-screen component styles... */
    </style>

    <!-- 3. Async-load non-critical CSS -->
    <link
      rel="preload"
      href="typography.css"
      as="style"
      onload="this.rel='stylesheet'"
    />
    <link
      rel="preload"
      href="layout.css"
      as="style"
      onload="this.rel='stylesheet'"
    />
    <link
      rel="preload"
      href="components.css"
      as="style"
      onload="this.rel='stylesheet'"
    />
    <link
      rel="preload"
      href="animations.css"
      as="style"
      onload="this.rel='stylesheet'"
    />

    <!-- 4. Print CSS: add media="print" so it doesn't block screen rendering -->
    <link rel="stylesheet" href="print.css" media="print" />

    <!-- 5. Google Fonts: async load -->
    <link
      rel="preload"
      href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700"
      as="style"
      onload="this.rel='stylesheet'"
    />

    <!-- Fallback for no-JS -->
    <noscript>
      <link rel="stylesheet" href="typography.css" />
      <link rel="stylesheet" href="layout.css" />
      <link rel="stylesheet" href="components.css" />
      <link rel="stylesheet" href="animations.css" />
      <link
        rel="stylesheet"
        href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700"
      />
    </noscript>
  </head>
  <body>
    ...
  </body>
</html>
```

**Issues fixed:**

1. `print.css` was render-blocking screens unnecessarily → `media="print"`
2. All CSS in `<link rel="stylesheet">` blocks rendering → use preload + async swap
3. No Google Fonts preconnect → DNS/TLS overhead on first font request
4. Font stylesheet itself was render-blocking → async loaded

</details>

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](./01-rendering-pipeline.md) — Where CSSOM fits in the full pipeline
- [`browser-internals/02-dom-tree-creation.md`](./02-dom-tree-creation.md) — DOM building (the other half of render tree)
- [`browser-internals/04-layout-reflow.md`](./04-layout-reflow.md) — What layout computes after CSSOM is ready
- [`browser-internals/08-critical-rendering-path.md`](./08-critical-rendering-path.md) — Optimizing the full path
- [`performance/03-layout-thrashing.md`](../performance/03-layout-thrashing.md) — How CSS reads cause layout thrashing

---

<div align="center">

**Next:** [`browser-internals/04-layout-reflow.md`](./04-layout-reflow.md) →

</div>
