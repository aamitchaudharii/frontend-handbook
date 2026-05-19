# 02 — DOM Tree Creation

> **"The DOM is not HTML. HTML is a text format. The DOM is a live, in-memory tree of objects that the browser builds from that text — and understanding how that tree is built explains why scripts block, why order matters, and why the browser can start rendering before your HTML finishes downloading."**

Every frontend engineer reads and modifies the DOM daily. Far fewer understand how it's actually created — the incremental parsing, the blocking behaviors, the tokenization pipeline, the tree construction algorithm. This document covers all of it: how bytes become nodes, what interrupts parsing, how the browser optimizes around blocking scripts, and what the finished tree actually looks like in memory.

---

## 📚 Table of Contents

1. [From Bytes to Tree — The Full Pipeline](#1-from-bytes-to-tree--the-full-pipeline)
2. [The HTML Tokenizer](#2-the-html-tokenizer)
3. [The Tree Construction Algorithm](#3-the-tree-construction-algorithm)
4. [Incremental Parsing — Why It Starts Before Download Finishes](#4-incremental-parsing--why-it-starts-before-download-finishes)
5. [Script Parsing and the Parser-Blocking Problem](#5-script-parsing-and-the-parser-blocking-problem)
6. [The Preload Scanner](#6-the-preload-scanner)
7. [`async` vs `defer` vs Module Scripts](#7-async-vs-defer-vs-module-scripts)
8. [CSS and Render Blocking](#8-css-and-render-blocking)
9. [What the DOM Tree Looks Like](#9-what-the-dom-tree-looks-like)
10. [DOM Node Types](#10-dom-node-types)
11. [The document Object and Document State](#11-the-document-object-and-document-state)
12. [DOMContentLoaded vs load](#12-domcontentloaded-vs-load)
13. [Performance Implications](#13-performance-implications)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. From Bytes to Tree — The Full Pipeline

The journey from a raw HTTP response to a usable DOM tree has five stages:

```
Network bytes (UTF-8 / UTF-16 encoded)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  STAGE 1: ENCODING DETECTION & BYTE CONVERSION       │
│  Read BOM or <meta charset>, convert bytes to chars  │
└─────────────────────────────┬───────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────┐
│  STAGE 2: TOKENIZATION                               │
│  Convert character stream into HTML tokens           │
│  (StartTag, EndTag, Character, Comment, DOCTYPE...)  │
└─────────────────────────────┬───────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────┐
│  STAGE 3: TREE CONSTRUCTION                          │
│  Process tokens → build the DOM tree                 │
│  Handle implicit open/close rules                    │
│  Manage the open elements stack                      │
└─────────────────────────────┬───────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────┐
│  STAGE 4: SCRIPT EXECUTION (interleaved)             │
│  Parser pauses when it encounters <script>           │
│  Script executes synchronously                       │
│  Parser resumes after script finishes                │
└─────────────────────────────┬───────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────┐
│  STAGE 5: COMPLETE DOM TREE                          │
│  DOMContentLoaded fires                              │
│  Render tree construction begins                     │
└─────────────────────────────────────────────────────┘
```

This entire pipeline runs **incrementally** as bytes stream in — the browser doesn't wait for the full HTML file before starting.

---

## 2. The HTML Tokenizer

The tokenizer reads the character stream and produces a sequence of **tokens** — the atomic units of HTML structure.

### Token Types

```
DOCTYPE token:       <!DOCTYPE html>
StartTag token:      <div class="container" id="app">
EndTag token:        </div>
Character token:     "Hello, World!" (text content)
Comment token:       <!-- This is a comment -->
EndOfFile token:     signals end of input
```

### Tokenizer State Machine

The tokenizer is a **state machine** with 80+ states defined in the HTML specification. A simplified view of the major states:

```
      ┌──────────────────────────────────────────────────┐
      │                   STATES                          │
      │                                                    │
      │  DATA  ──'<'──►  TAG_OPEN                         │
      │                      │                            │
      │                  '/' or letter                    │
      │                      │                            │
      │         ┌────────────┴──────────────┐             │
      │         ▼                           ▼             │
      │    END_TAG_OPEN              TAG_NAME             │
      │    (reads tag name)          (reads tag name)     │
      │         │                           │             │
      │         │                    space or '/'         │
      │         │                           │             │
      │         │                    BEFORE_ATTRIBUTE_NAME│
      │         │                           │             │
      │         ▼                           ▼             │
      │    END TAG TOKEN              START TAG TOKEN     │
      └──────────────────────────────────────────────────┘
```

### Error Tolerance

The HTML parser is intentionally **error-tolerant**. Unlike XML, malformed HTML is not an error — the parser has defined rules for handling every broken case:

```html
<!-- Missing closing tag — parser inserts it implicitly -->
<div>
  <p>First
  <p>Second
</div>

<!-- Parser sees this as: -->
<div>
  <p>First</p>   ← implicitly closed when next <p> opens
  <p>Second</p>  ← implicitly closed by </div>
</div>
```

This error tolerance is why billions of broken HTML pages on the internet still display correctly — every browser follows the same recovery rules from the spec.

---

## 3. The Tree Construction Algorithm

As tokens arrive from the tokenizer, the **tree constructor** builds the DOM. It maintains a stack called the **open elements stack**.

### The Open Elements Stack

```
Processing: <html><head><title>My Page</title></head><body><p>Hello</p></body></html>

After <html>:
  Stack: [html]
  Tree: html

After <head>:
  Stack: [html, head]
  Tree: html → head

After <title>:
  Stack: [html, head, title]
  Tree: html → head → title

After "My Page" (text token):
  Stack: [html, head, title]
  Tree: html → head → title → "My Page" (text node)

After </title>:
  Stack: [html, head]    ← title popped
  Tree: html → head → title → "My Page"

After </head>:
  Stack: [html]          ← head popped

After <body>:
  Stack: [html, body]
  Tree: html → head, body

After <p>:
  Stack: [html, body, p]
  Tree: html → head, body → p

After "Hello":
  Stack: [html, body, p]
  Tree: html → head, body → p → "Hello" (text node)

After </p>:
  Stack: [html, body]    ← p popped

After </body>:
  Stack: [html]

After </html>:
  Stack: []              ← done

FINAL TREE:
  document
    └── html
          ├── head
          │    └── title
          │          └── #text "My Page"
          └── body
                └── p
                      └── #text "Hello"
```

### Implicit Element Insertion

The tree constructor inserts elements that are required by the spec but missing from the source:

```html
<!-- Source HTML (no html/head/body tags) -->
<title>My Page</title>
<p>Hello</p>

<!-- Browser builds: -->
document └── html ← implicitly inserted ├── head ← implicitly inserted │ └──
title → "My Page" └── body ← implicitly inserted └── p → "Hello"
```

### Adoption Agency Algorithm

When a tag is improperly nested, the **Adoption Agency Algorithm** restructures the tree to produce valid nesting:

```html
<!-- Source: improperly nested formatting -->
<b><i>bold italic</b></i>

<!-- After Adoption Agency Algorithm: -->
<b><i>bold italic</i></b>
<!-- The </b> triggers restructuring to produce valid nesting -->
```

---

## 4. Incremental Parsing — Why It Starts Before Download Finishes

The HTML parser is **streaming** — it processes bytes as they arrive from the network, without waiting for the complete HTML file.

```
Network delivers:

t=0ms:   <html><head><link rel="stylesheet" href="styles.css">
          ↑ browser dispatches request for styles.css immediately

t=50ms:  <script src="app.js"></script>
          ↑ browser dispatches request for app.js

t=100ms: </head><body><div id="app">
          ↑ browser starts building render tree for visible elements

t=200ms: <h1>Hello World</h1>
          ↑ browser can render this heading — HTML not fully received yet!

t=400ms: </body></html>  ← HTML fully received
```

### Why Incremental Parsing Is Powerful

Without incremental parsing:

- Users see a blank page until the entire HTML downloads
- A 1MB HTML page on a slow connection = 5+ seconds of blank screen

With incremental parsing:

- Browser starts rendering above-the-fold content while the rest downloads
- Network requests for CSS/JS/images begin immediately on discovery
- First Contentful Paint can happen in milliseconds

### Parsing Is Single-Threaded

Parsing HTML runs on the **main thread**. This is important because:

```
Main thread during parsing:

[Parse HTML] [Execute script] [Parse more HTML] [Execute inline script] [Parse rest]
     ↑               ↑                                   ↑
  bytes arrive   script tag found               bytes arrive
                 parser PAUSES
```

Any script that executes during parsing runs on the same thread — and can **modify the DOM** that the parser is building. This is why the parser must stop and wait.

---

## 5. Script Parsing and the Parser-Blocking Problem

A `<script>` tag without `async` or `defer` is **parser-blocking** — it halts HTML parsing until the script downloads and executes.

### Why Scripts Block the Parser

Scripts can call `document.write()` which injects new HTML into the parse stream. The parser can't skip ahead — the injected HTML might change what comes next. So it must stop, wait for the script to download and execute, then resume.

```
Timeline with a blocking script:

HTML parsing:    ████████░░░░░░░░░░░░░░░░░░░░████████████████
                         ↑                  ↑
                   script tag found    script executed
                   parser pauses       parser resumes

Script download:          ░░░░░░░░░░░
Script execution:                     ░░░░░

Total delay: download time + execution time
```

### The Real-World Impact

```html
<!-- ❌ Script in <head> — delays everything below it -->
<head>
  <script src="heavy-analytics.js"></script>
  <!-- parser blocked here for 200ms -->
  <!-- user sees blank page during this time -->
</head>
<body>
  <h1>Hello</h1>
  <!-- can't render until script finishes -->
</body>
```

```html
<!-- ✅ Script at bottom of <body> — HTML parsed and rendered first -->
<body>
  <h1>Hello</h1>
  <!-- rendered immediately -->
  <!-- rest of content... -->
  <script src="heavy-analytics.js"></script>
  <!-- script loads AFTER content is visible -->
</body>
```

### Inline Scripts vs External Scripts

```html
<!-- Inline script: no download wait, but still blocks parsing -->
<script>
  // executes immediately, synchronously
  // parser blocked until this completes
  heavySynchronousOperation(); // bad if slow
</script>

<!-- External script: download + execute, blocks parsing -->
<script src="external.js"></script>
```

---

## 6. The Preload Scanner

The browser has a secret weapon against script-blocking: the **preload scanner** (also called the speculative parser).

### How It Works

When the main HTML parser is blocked by a script, a secondary lightweight scanner reads **ahead** in the HTML source and dispatches resource download requests:

```
Main parser: blocked at <script src="app.js">
                    waiting for app.js to download and execute

Preload scanner: reads ahead while main parser waits
  → finds <link rel="stylesheet" href="styles.css">  → dispatches download
  → finds <img src="hero.jpg">                       → dispatches download
  → finds <script src="analytics.js">                → dispatches download

When main parser unblocks:
  styles.css, hero.jpg, analytics.js are already downloading
  (or may even be ready) — no additional wait
```

### What the Preload Scanner Finds

```html
<head>
  <script src="blocking.js"></script>
  ← main parser blocked here
  <!-- preload scanner discovers these while waiting: -->
  <link rel="stylesheet" href="styles.css" />
  ← ✅ preloaded
  <link rel="preload" href="font.woff2" as="font" />
  ← ✅ preloaded
</head>
<body>
  <img src="hero.jpg" /> ← ✅ preloaded
  <script src="analytics.js"></script>
  ← ✅ preloaded (but not executed yet)
</body>
```

### What the Preload Scanner Misses

The preload scanner only reads the raw HTML source. It cannot see:

```javascript
// ❌ Resources injected by JavaScript — preload scanner misses these
const link = document.createElement("link");
link.rel = "stylesheet";
link.href = "/dynamic-styles.css";
document.head.appendChild(link); // discovered only when script runs

// ❌ CSS background images — preload scanner misses these
// (not in HTML source — defined in stylesheets)
```

This is why `<link rel="preload">` exists — to explicitly tell the preload scanner about resources it would otherwise miss.

---

## 7. `async` vs `defer` vs Module Scripts

These attributes change when scripts download and execute relative to HTML parsing.

### Comparison Table

```
Attribute    Download        Execution               Blocks Parser?
──────────────────────────────────────────────────────────────────
(none)       Immediately     As soon as downloaded   YES — immediately
async        Immediately     As soon as downloaded   YES — when ready
             (parallel)      (may be mid-parse)
defer        Immediately     After HTML fully parsed  NO
             (parallel)      In document order
type=module  Immediately     After HTML fully parsed  NO
             (parallel)      In document order
             (deferred by    (like defer)
             default)
```

### Visual Timeline

```
HTML:     ████████████████████████████████████████████████

(none):   DL─────────────────── EX parse resumes

async:    DL──────────── (parallel) ── EX parse resumes
          (whenever DL finishes, EX happens — could be mid-parse)

defer:    DL──────────── (parallel)
          parse continues ─────────────────── EX (after parse complete)

module:   DL──────────── (parallel) [deep deps too]
          parse continues ─────────────────── EX (after parse complete)
```

### When to Use Each

```javascript
// ❌ No attribute: only acceptable for scripts that must run at a specific
//    parse point (legacy code, document.write users)
<script src="legacy.js"></script>

// ✅ async: scripts with no dependencies on DOM or other scripts
//    (analytics, ads, monitoring)
<script async src="analytics.js"></script>

// ✅ defer: most application scripts — respects document order,
//    DOM is ready when they execute
<script defer src="app.js"></script>
<script defer src="components.js"></script> <!-- runs after app.js -->

// ✅ type="module": modern ES module scripts — deferred by default,
//    strict mode, isolated scope
<script type="module" src="main.js"></script>
```

### The `DOMContentLoaded` Connection

```
No attribute:  DOMContentLoaded fires AFTER all blocking scripts execute
async:         DOMContentLoaded fires independently (may be before or after async scripts)
defer:         DOMContentLoaded fires AFTER all defer scripts execute
               (defer scripts run just before DOMContentLoaded)
type=module:   same as defer
```

---

## 8. CSS and Render Blocking

CSS doesn't block HTML **parsing** — but it blocks HTML **rendering**. And it can indirectly block script execution.

### Why CSS Is Render-Blocking

The browser cannot display any content until it has a complete CSSOM. The reason: CSS is cascading — a single rule at the bottom of a stylesheet can affect the styling of elements at the top. The browser can't safely render anything until it knows all the rules.

```
HTML parsing:  ████████████████████████████████████████
CSS parsing:   ░░░░░░░░░░░░░░░░░░░░░░░
                ↑                    ↑
         CSS request                CSSOM
         dispatched                 complete

Render tree:                         ████████████████
                                     ↑
                             (must wait for CSSOM)

First paint:                              ↑
                             (must wait for render tree)
```

### CSS Blocking Script Execution

There's a subtle interaction: if a CSS file is downloading AND a `<script>` tag appears in the HTML after it, **the script waits for the CSS before executing**:

```html
<link rel="stylesheet" href="slow-styles.css" />
<!-- starts downloading -->
<script>
  // Browser waits for slow-styles.css before running this
  // Why? Because scripts can call getComputedStyle() which needs CSSOM
  const color = getComputedStyle(document.body).color;
</script>
```

This means a slow stylesheet can delay script execution, which delays the rest of HTML parsing.

### Minimizing Render-Blocking CSS

```html
<!-- ✅ Critical CSS inlined — no blocking request needed -->
<style>
  /* Above-the-fold styles only */
  body {
    margin: 0;
    font-family: sans-serif;
  }
  .hero {
    background: navy;
    color: white;
    padding: 2rem;
  }
</style>

<!-- Non-critical CSS loaded asynchronously -->
<link
  rel="preload"
  href="full-styles.css"
  as="style"
  onload="this.onload=null;this.rel='stylesheet'"
/>
<noscript><link rel="stylesheet" href="full-styles.css" /></noscript>
```

---

## 9. What the DOM Tree Looks Like

The finished DOM tree is not just a mirror of your HTML structure. It includes nodes for whitespace, comments, and processing instructions — and it has a specific hierarchy.

### Full Tree Structure

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Example</title>
  </head>
  <body>
    <!-- main content -->
    <main>
      <h1>Hello</h1>
      <p>World</p>
    </main>
  </body>
</html>
```

```
document (Document node)
│
├── DOCTYPE: html (DocumentType node)
│
└── html (Element node, attributes: lang="en")
      │
      ├── #text "\n  " (Text node — whitespace between html and head)
      │
      ├── head (Element node)
      │    ├── #text "\n    " (whitespace)
      │    ├── meta (Element node, attributes: charset="UTF-8")
      │    ├── #text "\n    " (whitespace)
      │    ├── title (Element node)
      │    │     └── #text "Example" (Text node)
      │    └── #text "\n  " (whitespace)
      │
      ├── #text "\n  " (whitespace between head and body)
      │
      └── body (Element node)
           ├── #text "\n    " (whitespace)
           ├── Comment: " main content " (Comment node)
           ├── #text "\n    " (whitespace)
           ├── main (Element node)
           │    ├── #text "\n      " (whitespace)
           │    ├── h1 (Element node)
           │    │    └── #text "Hello"
           │    ├── #text "\n      " (whitespace)
           │    ├── p (Element node)
           │    │    └── #text "World"
           │    └── #text "\n    " (whitespace)
           └── #text "\n  " (whitespace)
```

**Key insight:** Whitespace between elements becomes text nodes. This is why `element.childNodes` contains more entries than `element.children` — `childNodes` includes text and comment nodes, `children` only includes Element nodes.

```javascript
// <ul>\n  <li>A</li>\n  <li>B</li>\n</ul>

ul.childNodes.length; // 5 (text, li, text, li, text — whitespace counts!)
ul.children.length; // 2 (only li elements)

ul.firstChild; // #text "\n  " (the whitespace text node)
ul.firstElementChild; // <li>A</li> (the first Element)
```

---

## 10. DOM Node Types

Every node in the DOM tree has a `nodeType` property. The most important:

```javascript
Node.ELEMENT_NODE = 1; // <div>, <p>, <span>...
Node.ATTRIBUTE_NODE = 2; // (deprecated for direct use)
Node.TEXT_NODE = 3; // "Hello World" text content
Node.CDATA_SECTION_NODE = 4; // <![CDATA[ ... ]]>
Node.PROCESSING_INSTRUCTION_NODE = 7; // <?xml version="1.0"?>
Node.COMMENT_NODE = 8; // <!-- comment -->
Node.DOCUMENT_NODE = 9; // the document object itself
Node.DOCUMENT_TYPE_NODE = 10; // <!DOCTYPE html>
Node.DOCUMENT_FRAGMENT_NODE = 11; // DocumentFragment
```

### Node Interface Hierarchy

```
EventTarget
  └── Node
        ├── Document
        │     └── HTMLDocument
        ├── DocumentFragment
        ├── DocumentType
        ├── CharacterData
        │     ├── Text
        │     ├── Comment
        │     └── CDATASection
        └── Element
              └── HTMLElement
                    ├── HTMLDivElement
                    ├── HTMLInputElement
                    ├── HTMLScriptElement
                    └── ... (one per HTML element type)
```

Every DOM element is an instance of this hierarchy. A `<div>` is an `HTMLDivElement` which extends `HTMLElement` which extends `Element` which extends `Node` which extends `EventTarget`.

```javascript
const div = document.createElement("div");

div instanceof HTMLDivElement; // true
div instanceof HTMLElement; // true
div instanceof Element; // true
div instanceof Node; // true
div instanceof EventTarget; // true

// All of these are the SAME object
// The prototype chain provides all the interfaces
```

---

## 11. The document Object and Document State

The `document` object is the root of the DOM tree and the entry point for all DOM operations. It has a **readyState** property that tracks parsing progress.

### Document Ready States

```javascript
document.readyState:
  'loading'      → HTML is still being parsed
  'interactive'  → HTML parsed, DOMContentLoaded about to fire,
                   subresources (images, CSS) still loading
  'complete'     → everything loaded (DOMContentLoaded AND load fired)
```

```javascript
// Listen for state transitions
document.addEventListener("readystatechange", () => {
  console.log("State:", document.readyState);
});

// Timeline:
// 'loading'     → as HTML parses
// 'interactive' → HTML parsed (DOMContentLoaded fires here)
// 'complete'    → all subresources loaded (load fires here)
```

### document.write() — The Parser's Nemesis

```javascript
// document.write() injects HTML directly into the parse stream
// This is why scripts block the parser — they might call this

document.write("<p>Injected content</p>");
// If called during parsing: inserts into parse stream
// If called after parsing (DOMContentLoaded or load): REPLACES entire document!
```

`document.write()` is deprecated and its use causes multiple problems:

- Forces the parser to stop
- Prevents async/defer on the calling script
- Can destroy the entire document if called after parsing
- Disabled for Service Worker-controlled pages

---

## 12. DOMContentLoaded vs load

Two critical timing events — frequently confused.

### DOMContentLoaded

Fires when:

- HTML has been fully parsed
- All synchronous and `defer` scripts have executed
- The DOM is fully built and accessible

Does NOT wait for:

- Images to load
- Stylesheets to load
- Iframes to load
- Fonts to load

```javascript
document.addEventListener("DOMContentLoaded", () => {
  // ✅ Safe to query DOM, attach listeners, initialize components
  const app = document.getElementById("app");
  initializeApp(app);
});
```

### load (window.onload)

Fires when:

- Everything DOMContentLoaded waited for
- PLUS all subresources: images, stylesheets, iframes, fonts

```javascript
window.addEventListener("load", () => {
  // ✅ Images, CSS, fonts all loaded
  // Good for: measuring page size, reading image dimensions,
  // initializing features that depend on layout being complete
  const imageHeight = document.querySelector(".hero img").naturalHeight;
});
```

### Timeline

```
HTML starts downloading
       │
       ▼
HTML parsing begins (incremental)
       │
       ├── CSS requests dispatched → CSSOM building
       ├── Script requests dispatched
       ├── Image requests dispatched
       │
       ▼ (all defer scripts executed, CSSOM ready)
DOMContentLoaded fires ← "DOM is ready"
       │
       ├── Images still loading
       ├── Fonts still loading
       │
       ▼ (all subresources complete)
window load fires ← "Everything is ready"
```

### Which to Use

```javascript
// ❌ Often too late — user waiting with blank screen
window.addEventListener("load", initApp);

// ✅ DOM is ready — initialize immediately
document.addEventListener("DOMContentLoaded", initApp);

// ✅ Even better — if script is at bottom of <body> or is defer/module
// DOM is already parsed when script executes
// No event listener needed
initApp(); // document is already interactive
```

---

## 13. Performance Implications

### Parsing Speed

Typical HTML parsing speed: **~3-12MB/second** on modern hardware. This means:

```
100KB HTML: ~8-33ms to parse
1MB HTML:   ~83-333ms to parse
10MB HTML:  ~833ms - 3.3s (severe)
```

Large HTML payloads degrade parsing performance even before any JavaScript runs.

### What Slows Parsing

```
1. Blocking scripts — pause parser, wait for download + execute
2. Large inline scripts — execute synchronously during parse
3. Large HTML size — more bytes to parse
4. Deep nesting — more complex tree construction
5. document.write() calls in scripts — inject more parsing work
```

### Measuring Parse Time

```javascript
// Use Navigation Timing API to measure parsing milestones
const timing = performance.getEntriesByType("navigation")[0];

console.log("DOM interactive (parsing done):", timing.domInteractive, "ms");
console.log("DOM complete (load done):", timing.domComplete, "ms");
console.log("Parse time:", timing.domInteractive - timing.responseStart, "ms");
```

### Resource Hints for Faster Discovery

```html
<!-- Tell the browser about resources it will need -->
<link rel="preload" href="/app.js" as="script" />
<link rel="preload" href="/styles.css" as="style" />
<link rel="preload" href="/hero.webp" as="image" />
<link rel="preconnect" href="https://api.example.com" />
<link rel="dns-prefetch" href="https://fonts.googleapis.com" />
```

---

## 14. Good Practices

### ✅ Place scripts at the bottom of `<body>` or use `defer`

```html
<!-- ✅ Content renders before scripts execute -->
<!DOCTYPE html>
<html>
  <head>
    <!-- CSS in head — render-blocking but expected -->
    <link rel="stylesheet" href="styles.css" />
  </head>
  <body>
    <main><!-- content --></main>

    <!-- Scripts at bottom: DOM is fully parsed before they run -->
    <script src="app.js"></script>
  </body>
</html>

<!-- OR: defer in head — same effect, cleaner HTML -->
<head>
  <script defer src="app.js"></script>
</head>
```

### ✅ Use `defer` instead of "scripts at bottom" for ES modules

```html
<!-- ✅ Modern approach: defer or module in head -->
<script defer src="app.js"></script>
<script type="module" src="main.js"></script>
<!-- Both: parallel download, execute after parsing -->
```

### ✅ Inline critical CSS, async-load non-critical CSS

```html
<!-- ✅ Critical CSS: inlined, no blocking request -->
<style>
  /* Minimal above-the-fold styles */
  body {
    margin: 0;
  }
  .hero {
    background: #001f3f;
  }
</style>

<!-- ✅ Full CSS: loaded asynchronously -->
<link rel="preload" href="full.css" as="style" onload="this.rel='stylesheet'" />
```

### ✅ Use `children` over `childNodes` for element iteration

```javascript
// ✅ children: only Element nodes (no whitespace text nodes)
for (const child of element.children) {
  processElement(child);
}

// ❌ childNodes: includes text nodes, comment nodes
// Requires filtering:
for (const node of element.childNodes) {
  if (node.nodeType === Node.ELEMENT_NODE) {
    // extra check needed
    processElement(node);
  }
}
```

### ✅ Use `DocumentFragment` for batch DOM insertion

```javascript
// ✅ Build off-DOM, insert once — one reflow
const fragment = document.createDocumentFragment();
items.forEach((item) => {
  const li = document.createElement("li");
  li.textContent = item.name;
  fragment.appendChild(li); // no layout triggered — fragment is off-DOM
});
list.appendChild(fragment); // ONE layout trigger
```

---

## 15. Bad Practices

### ❌ `document.write()` in scripts

```html
<!-- ❌ Forces synchronous parsing, blocks rendering, deprecated -->
<script>
  document.write('<link rel="stylesheet" href="extra.css">');
  document.write("<div>Injected content</div>");
</script>
```

### ❌ Parser-blocking scripts in `<head>` without defer/async

```html
<!-- ❌ User sees blank page until these download and execute -->
<head>
  <script src="framework.js"></script>
  <!-- 400KB, takes 2s -->
  <script src="app.js"></script>
  <!-- waits for framework.js -->
</head>
<body>
  <!-- nothing renders until both scripts finish -->
</body>
```

### ❌ Using `document.readyState === 'complete'` for DOM access

```javascript
// ❌ You don't need to wait for full load just to access the DOM
if (document.readyState === "complete") {
  initApp(); // waiting for images to load unnecessarily
}

// ✅ DOMContentLoaded is sufficient for DOM access
document.addEventListener("DOMContentLoaded", initApp);
```

### ❌ Building large DOM trees in JavaScript unnecessarily

```javascript
// ❌ Creating 10,000 DOM nodes via JavaScript is slow
// Consider: virtualization, server-side HTML, or canvas
const list = document.getElementById("list");
data.forEach((item) => {
  list.innerHTML += `<li>${item.name}</li>`; // VERY slow - re-parses entire list each time
});
```

---

## 16. Common Mistakes

### Mistake 1 — Thinking DOM equals HTML

```javascript
// HTML source:
// <p>Hello World</p>

// DOM has MORE than the HTML shows:
document.body.childNodes; // may include whitespace text nodes
document.body.firstChild; // might be a text node "\n", not <p>
document.body.firstElementChild; // this is what you usually want: <p>
```

### Mistake 2 — Not accounting for DOMContentLoaded timing

```javascript
// ❌ Script in <head>, no defer — DOM not ready
document.getElementById("app").textContent = "Hello"; // null reference!
// #app doesn't exist yet — parser hasn't reached it

// ✅ Wait for DOMContentLoaded, or use defer
document.addEventListener("DOMContentLoaded", () => {
  document.getElementById("app").textContent = "Hello"; // ✅
});
```

### Mistake 3 — Conflating `async` and `defer`

```javascript
// async: executes AS SOON as it downloads (may interrupt parsing)
// Good for: truly independent scripts (analytics)
// BAD for: scripts that depend on each other or on DOM

// defer: executes AFTER all HTML is parsed, IN ORDER
// Good for: app scripts that need DOM and have dependencies
// Two defer scripts: guaranteed to execute in document order
```

### Mistake 4 — `innerHTML` for reading vs writing

```javascript
// Reading innerHTML re-serializes the DOM to HTML string — slow
for (const child of element.children) {
  if (child.innerHTML.includes("hello")) {
    /* ... */
  } // ❌ re-serializes each time
}

// ✅ Access text content directly
for (const child of element.children) {
  if (child.textContent.includes("hello")) {
    /* ... */
  } // ✅ direct property
}
```

---

## 17. Interview-Level Explanation

> **"How does the browser build the DOM? What causes parsing to block?"**

**Strong answer:**

> "The browser builds the DOM incrementally as HTML bytes stream in from the network — it doesn't wait for the complete file. The process has two main stages: tokenization, which converts the character stream into HTML tokens (start tags, end tags, text, comments), and tree construction, which processes those tokens to build the node tree using an open elements stack.
>
> The parser is intentionally error-tolerant — malformed HTML follows defined recovery rules rather than throwing errors, which is why broken HTML still renders consistently across browsers.
>
> Scripts are the main cause of parse blocking. A `<script>` tag without `async` or `defer` halts the parser entirely until the script downloads and executes. This is because scripts can call `document.write()`, which injects HTML directly into the parse stream — so the parser must let the script run before it knows what comes next.
>
> The browser's answer to this is the preload scanner — a secondary thread that reads ahead in the HTML while the main parser is blocked, dispatching download requests for CSS, images, and other scripts so they can load in parallel with the blocking script.
>
> `defer` scripts download in parallel but execute after the full HTML is parsed, in document order. `async` scripts download in parallel and execute as soon as they're ready, potentially mid-parse. For most application code, `defer` is the right choice.
>
> CSS doesn't block HTML parsing, but it blocks rendering — the browser won't display anything until the CSSOM is complete, because a single CSS rule could affect any element already in the tree. CSS also indirectly blocks script execution: if a stylesheet is downloading when a `<script>` tag is encountered, the browser waits for the CSS before running the script, because scripts might call `getComputedStyle()`."

---

## 18. Exercises

### Exercise 1 — Identify parse-blocking elements

```html
<!DOCTYPE html>
<html>
  <head>
    <link rel="stylesheet" href="main.css" />
    <script src="analytics.js"></script>
    <script defer src="app.js"></script>
    <script async src="widget.js"></script>
  </head>
  <body>
    <h1>Hello</h1>
    <script>
      console.log(document.querySelector("h1"));
    </script>
    <p>World</p>
  </body>
</html>
```

For each script/link, answer:

1. Does it block HTML parsing?
2. Does it block rendering?
3. When does it execute relative to HTML parsing?
4. What is the output of the inline script?

<details>
<summary>Answers</summary>

```
main.css (link stylesheet):
  - Blocks rendering: YES (CSSOM must be ready before render tree)
  - Blocks parsing: NO (CSS doesn't block parsing)
  - Blocks analytics.js execution: YES (script waits for CSS before running)

analytics.js (no attribute):
  - Blocks parsing: YES — parser pauses, downloads, executes, then resumes
  - Execution: synchronously, as soon as downloaded, mid-parse in <head>
  - Note: also waits for main.css to finish first (CSS before script rule)

app.js (defer):
  - Blocks parsing: NO — downloads in parallel
  - Execution: AFTER full HTML parsed, before DOMContentLoaded
  - In order with other defer scripts

widget.js (async):
  - Blocks parsing: partially — downloads in parallel, but executes
    whenever it's ready (could interrupt parsing)
  - Execution: non-deterministic timing relative to parsing

Inline script in <body>:
  console.log(document.querySelector('h1'))
  → <h1>Hello</h1> element IS in the DOM at this point
  → Parser has processed <h1>Hello</h1> before reaching this script
  → Output: <h1> element (not null)
```

</details>

---

### Exercise 2 — Count the DOM nodes

How many DOM nodes does this HTML create? (Include ALL node types.)

```html
<ul>
  <li>First</li>
  <li>Second</li>
</ul>
```

<details>
<summary>Answer</summary>

```
Nodes created:

1. <ul>          (Element)
2. "\n  "        (Text — whitespace before first <li>)
3. <li>          (Element)
4. "First"       (Text)
5. "\n  "        (Text — whitespace between </li> and <li>)
6. <li>          (Element)
7. "Second"      (Text)
8. "\n"          (Text — whitespace before </ul>)

Total: 8 nodes

ul.childNodes.length  → 5  (whitespace + li + whitespace + li + whitespace)
ul.children.length    → 2  (only the two <li> elements)
```

</details>

---

### Exercise 3 — Script loading order

```html
<head>
  <script defer src="A.js"></script>
  <script async src="B.js"></script>
  <script defer src="C.js"></script>
</head>
```

Assuming network conditions:

- A.js downloads in 300ms
- B.js downloads in 100ms
- C.js downloads in 200ms

In what order do they execute?

<details>
<summary>Answer</summary>

```
B.js executes first: async, downloads in 100ms, executes immediately when ready
  → May execute while HTML is still parsing (100ms in)

C.js executes second: defer, downloads in 200ms (before A.js at 300ms)
  → BUT defer guarantees DOCUMENT ORDER
  → Even though C.js is ready before A.js, it waits for A.js

A.js executes third: defer, downloads in 300ms
  → A.js and C.js run in document order (A then C) after parsing completes

Final execution order: B → A → C

Key insight: async ignores document order (whenever ready).
            defer preserves document order (A before C, regardless of download speed).
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/01-rendering-pipeline.md`](./01-rendering-pipeline.md) — What happens after DOM is built
- [`browser-internals/03-cssom.md`](./03-cssom.md) — CSSOM construction in depth
- [`browser-internals/08-critical-rendering-path.md`](./08-critical-rendering-path.md) — Optimizing first paint
- [`performance/01-dom-optimization.md`](../performance/01-dom-optimization.md) — Working efficiently with the DOM
- [`rendering/01-dom-batching.md`](../rendering/01-dom-batching.md) — Batching DOM mutations

---

<div align="center">

**Next:** [`browser-internals/03-cssom.md`](./03-cssom.md) →

</div>
