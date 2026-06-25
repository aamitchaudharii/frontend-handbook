# 01 — Chrome DevTools

> **"DevTools is not a viewer — it's a laboratory. You don't just watch what's happening; you pause it, inspect its internals, modify it live, replay it, and measure it. The difference between a developer who finds bugs in minutes and one who spends hours is usually fluency with this laboratory."**

Chrome DevTools is the most comprehensive frontend debugging environment available. Most developers use 20% of its capabilities and wonder why bugs take so long to find. This document covers the full range: Elements for DOM/CSS inspection and live modification, Sources for JavaScript debugging with breakpoints and call stacks, Network for request analysis and latency, Performance for profiling and flame charts, Memory for leak detection, and Application for storage and Service Worker inspection.

---

## 📚 Table of Contents

1. [Elements Panel — DOM and CSS Debugging](#1-elements-panel--dom-and-css-debugging)
2. [Sources Panel — JavaScript Debugging](#2-sources-panel--javascript-debugging)
3. [Breakpoint Types](#3-breakpoint-types)
4. [Call Stack and Scope Inspection](#4-call-stack-and-scope-inspection)
5. [Network Panel — Request Analysis](#5-network-panel--request-analysis)
6. [Performance Panel — Profiling](#6-performance-panel--profiling)
7. [Memory Panel — Leak Detection](#7-memory-panel--leak-detection)
8. [Application Panel — Storage and Service Workers](#8-application-panel--storage-and-service-workers)
9. [Console Panel — Advanced Usage](#9-console-panel--advanced-usage)
10. [Rendering Panel — Visual Diagnostics](#10-rendering-panel--visual-diagnostics)
11. [DevTools Shortcuts](#11-devtools-shortcuts)
12. [Good Practices](#12-good-practices)
13. [Common Debugging Workflows](#13-common-debugging-workflows)
14. [Interview-Level Explanation](#14-interview-level-explanation)

---

## 1. Elements Panel — DOM and CSS Debugging

### DOM Inspection

```
KEY OPERATIONS:

Inspect Element:
  Right-click → Inspect | Ctrl+Shift+C (toggle inspect mode)
  Click element in page → jumps to DOM node in Elements panel

Edit HTML:
  Double-click text → edit inline
  Double-click attribute → edit attribute value
  Right-click node → Edit as HTML (entire node, including children)

DOM Breakpoints:
  Right-click element → Break on →
    "subtree modifications" — fires when any descendant changes
    "attribute modifications" — fires when any attribute changes
    "node removal" — fires when this node is removed from DOM
  INVALUABLE for: "why does this element disappear?" or "what's changing this attribute?"

Force Element States:
  Right-click element → Force state → :hover, :active, :focus, :focus-within, :visited
  OR: in Styles panel, click ":hov" button at top
  Lets you inspect styles for states without holding the mouse/keyboard

$0 in Console:
  The currently selected element in Elements panel is $0 in Console
  $0.className, $0.getBoundingClientRect(), $0.style.transform, etc.
```

### CSS Debugging

```
STYLES PANEL:
  Shows all CSS rules affecting the selected element
  Struck-through: overridden or invalid property
  Orange value: property computed differently than written (e.g., auto)
  Orange !important: overridden by higher-specificity rule

  Live editing:
  Click any value → type new value → Enter to apply immediately
  Click checkbox next to property → toggle on/off
  Click "+" → add new CSS rule
  Click empty space in rule → add new property

  Filter: type in filter box → find which rule sets a specific property
  "Computed" tab → see final computed values (resolves inheritance, cascade)

LAYOUT PANEL:
  Grid overlay: select grid container → "Grid" section shows grid lines overlay
  Flexbox overlay: same for flex containers
  Box model diagram: margin, border, padding, content dimensions
  Click any value in box model → edit inline

SOURCES → COVERAGE:
  More tools → Coverage → Start → reload page
  Shows unused CSS (red bars) vs used CSS (green bars)
  Identify dead CSS that could be removed
```

---

## 2. Sources Panel — JavaScript Debugging

```
LAYOUT:
  Left panel: file tree (navigate source files)
  Center panel: source code viewer/editor
  Right panel: debugger controls (pause/step buttons, scope, call stack, etc.)

NAVIGATION:
  Ctrl+P (Cmd+P Mac): open file by name (fuzzy search)
  Ctrl+Shift+P: command palette (run any DevTools command)
  Ctrl+F: search in current file
  Ctrl+Shift+F: search across all files

LIVE CODE EDITING:
  You can edit source code directly in DevTools Sources panel
  Changes take effect immediately for inline scripts
  For bundled files: changes are in memory only, lost on reload
  For .js files served directly: Ctrl+S saves to file system (with Local Overrides)
```

### Local Overrides (Persistent Source Edits)

```
Sources → Overrides tab → Enable Local Overrides → select a folder
Now: edits to files in DevTools persist across reloads
Use case: test CSS/JS changes without modifying source code
          debug production minified files with readable versions
```

---

## 3. Breakpoint Types

### Regular Breakpoints

```javascript
// Click line number in Sources → blue dot = breakpoint
// Execution pauses when that line is about to execute

// Conditional breakpoint:
// Right-click line number → Add conditional breakpoint
// → only pauses when the condition evaluates to truthy
// e.g.: user.id === '42' || item.price > 1000

// Logpoint (non-pausing, logs to console):
// Right-click line number → Add logpoint
// → logs message when line executes, doesn't pause
// Great for: debugging tight loops or high-frequency events without stopping execution
// e.g.: 'User clicked: {event.target.className}'
```

### Debugger Statement

```javascript
// Add to code: execution pauses here when DevTools is open
function suspiciousFunction(data) {
  debugger; // ← pauses here in DevTools
  const processed = transform(data);
  return processed;
}
// Remove before committing to production!
```

### DOM Breakpoints (from Elements Panel)

```
Right-click DOM node → Break on →
  Subtree Modifications: fires when any descendant is added/removed/moved
  Attribute Modifications: fires when any attribute changes (class, style, data-*)
  Node Removal: fires when this node is removed

WHEN TO USE:
  "Something is changing this element's class and I don't know what" → attribute breakpoint
  "This element disappears and I can't find why" → node removal breakpoint
  "The DOM is being modified but I can't find which code does it" → subtree breakpoint
```

### Event Listener Breakpoints

```
Sources panel → right panel → Event Listener Breakpoints
Expand any category (Mouse, Keyboard, XHR, etc.)
Check a specific event type → pauses whenever that event fires

Useful:
  XHR Breakpoints → breaks on every fetch/XHR request
    Can filter by URL substring: right-click "XHR Breakpoints" → add URL pattern
  Keyboard → keydown/keyup → pause on specific keyboard events
  Timer → setTimeout/setInterval → pause when timer fires
```

### Exception Breakpoints

```
Sources panel → right panel → Pause on exceptions (hexagon icon)
Toggle "Pause on caught exceptions" too

Breaks execution at the moment an exception is thrown — BEFORE it's caught
Shows the exact line that threw, with the stack trace in call stack panel

INVALUABLE for: "I see an error in the console but I can't find where it's thrown"
Without this: error appears in console with a stack trace (sometimes minified)
With this: execution pauses AT the throw site with full variable context
```

---

## 4. Call Stack and Scope Inspection

```
WHEN PAUSED AT A BREAKPOINT:

CALL STACK panel (right side):
  Shows the function call chain that led to this point
  Click any frame → jumps to that code, scope updates to that frame's context
  Right-click frame → "Restart frame" → re-executes from start of that function
    (variables reset — useful for re-running without refreshing)

SCOPE panel:
  Local: variables in the current function scope
  Closure: captured variables from enclosing functions
  Global: window object properties

  Hover any value → tooltip shows the value
  Click disclosure triangle → expand objects/arrays
  Double-click a value → edit it live (changes take effect immediately)
  Use "Watch" panel → add expressions to watch across steps

STEPPING CONTROLS:
  F8 (Resume): continue execution until next breakpoint
  F10 (Step over): execute current line, pause at next line (don't enter function calls)
  F11 (Step into): enter the function call on current line
  Shift+F11 (Step out): complete current function, pause at its caller
  F9 (Step): like step-over but also steps into async boundaries
```

### The "Restart Frame" Technique

```javascript
// Scenario: you're paused in a function but missed something important earlier

// Stack:
// processData  ← you're here, paused on line 45
// handleSubmit
// React.dispatchEvent

// Right-click "processData" → "Restart frame"
// → processData re-executes from its first line
// → you can now step through from the beginning with full variable access
// → no need to reload the page and reproduce the bug trigger

// Limitation: external side effects (network calls, DOM changes) from the first
// execution remain — only the function's local state resets
```

---

## 5. Network Panel — Request Analysis

### Reading the Waterfall

```
WATERFALL COLUMNS (hover column header → select which to show):
  Status:     HTTP status code (200, 304, 404, etc.)
  Type:       document, script, stylesheet, fetch, xhr, img, font, ws
  Initiator:  which line of code triggered this request
  Size:       transferred size / uncompressed size
  Time:       total request time
  Waterfall:  visual timeline

WATERFALL COLOR BARS:
  DNS Lookup (dark green): domain name resolution
  TCP Connection (orange): TCP handshake
  TLS Negotiation (purple): HTTPS handshake
  Request Sent (brown): sending the request
  Waiting (green): server processing time (TTFB)
  Content Download (blue): downloading the response body

QUICK DIAGNOSTICS:
  Large "Waiting" bar: slow server response time
  Many sequential requests: potential request waterfall to fix
  304 responses: conditional requests (ETag/Last-Modified working)
  No DNS/TCP bars: connection was reused (good!)
  Red status: error response (4xx, 5xx)
```

### Network Panel Filters and Features

```
Filter bar:
  Type: filter by resource type (JS, CSS, IMG, Fetch, etc.)
  Search: filter by URL substring or regex
  "is:from-cache": show only cached responses
  -domain:example.com: exclude specific domain
  larger-than:100k: show resources over 100KB

Request details (click any request):
  Headers: request and response headers
  Preview: formatted preview (JSON, image, HTML)
  Response: raw response body
  Initiator: call stack that triggered the request
  Timing: detailed breakdown of request phases

Throttling (connection simulation):
  Dropdown at top → "Slow 3G", "Fast 3G", "Offline"
  Custom: add your own throttle profile
  Critical for testing on real-world network conditions

Preserve log: keep requests across page navigations
  Useful for: debugging redirect chains, SPA navigation issues
```

### Request Blocking

```
Right-click any request → "Block request URL" | "Block request domain"
Use: test what happens when a third-party script fails to load
     test fallback behavior when API is down
     find which resource is causing a layout shift
```

---

## 6. Performance Panel — Profiling

### Recording a Profile

```
Performance → Record → Interact with page → Stop

WHAT TO LOOK FOR:

FPS at top: red/yellow bars → dropped frames
  Red bar = 1+ dropped frames (noticeable jank)

CPU usage: high sustained CPU → main thread bottleneck

Flame chart (bottom):
  Each bar = a function call
  Width = time spent in that function (including callees)
  Color:
    Yellow = JavaScript
    Purple = Style/Layout (recalculate style, layout)
    Green  = Paint
    Blue   = Network/HTML parsing

Long Tasks:
  Red diagonal hatching on a task = Long Task (> 50ms)
  Click to zoom into that task and see what's inside

Call tree / Bottom-up / Event log tabs:
  Call tree: top-down view of all execution
  Bottom-up: which functions consumed the most SELF time
  Event log: chronological list of all events
```

### Reading the Flame Chart

```
EXAMPLE BOTTLENECK ANALYSIS:

1. Zoom into a Long Task (Shift+scroll or use timeline selector)
2. Identify the widest bar in the JS section
3. That bar's name = the most expensive function in this task
4. Check its children (below it in the chart) to find the actual culprit
5. Click the bar → "Reveal in Sources" to jump to the code

COMMON PATTERNS:
  Wide yellow bar for "recalculate style" after JS: style invalidation during animation
  Many short yellow bars in a loop: n × DOM operations (layout thrashing)
  Long time in "XHR Load": slow API response blocking the task
  Long "Parse HTML": large HTML document, consider streaming
  Long "Compile Script": large JS bundle, consider code splitting
```

---

## 7. Memory Panel — Leak Detection

```
HEAP SNAPSHOT:
  Memory → Heap snapshot → Take snapshot

  View options:
    Summary: grouped by constructor (most useful)
    Comparison: diff between two snapshots (find what leaked)
    Containment: full object graph (find retainers)
    Statistics: memory breakdown by type

FINDING A LEAK (Comparison view):
  1. Take snapshot 1 (baseline)
  2. Do the leaking action 5 times (e.g., open/close modal)
  3. Force GC (trash icon)
  4. Take snapshot 2
  5. Select snapshot 2, choose "Comparison" in dropdown
  6. Sort by "# Delta" — growing objects are potential leaks
  7. Click a growing type → see instances in bottom panel
  8. Click an instance → "Retainers" shows reference chain keeping it alive

DETACHED DOM NODES:
  In any heap snapshot: filter by "Detached"
  Shows DOM nodes removed from tree but still referenced in JS
  Click → Retainers shows which variable holds the reference
```

---

## 8. Application Panel — Storage and Service Workers

```
STORAGE (left sidebar):
  Local Storage / Session Storage:
    Click domain → see all key-value pairs
    Double-click value → edit inline
    Right-click → delete key

  IndexedDB:
    Browse database structure, object stores, and records
    Start transaction, modify data

  Cookies:
    See all cookies for the domain
    Filter, delete, add cookies
    See: name, value, domain, path, expiration, flags (HttpOnly, Secure, SameSite)

SERVICE WORKERS:
  See registered Service Workers
  "Update on reload": force SW update on every page reload (dev mode)
  "Bypass for network": skip SW cache for all requests (test without SW)
  "Offline" checkbox: simulate offline (SW must handle this gracefully)
  "Update" button: force check for SW update
  Inspect SW script: click the script link → opens in Sources

CACHE STORAGE:
  See all caches created by Service Worker
  Browse cached request/response pairs
  Delete individual entries or entire caches
  Verify caching strategies are working

MANIFEST (PWA):
  View web app manifest
  Check if PWA requirements are met (installability criteria)
  "Add to homescreen" button for testing PWA installation
```

---

## 9. Console Panel — Advanced Usage

```javascript
// SELECTION SHORTCUTS:
$("#id"); // querySelector('#id') — alias for document.querySelector
$$(".class"); // querySelectorAll('.class') — returns array (not NodeList)
$0; // currently selected element in Elements panel
($1, $2, $3, $4); // previously selected elements (history)
$_; // result of last expression evaluated in console

// MONITORING:
monitor(fn); // logs whenever fn is called (with arguments)
monitorEvents($0); // logs all events on selected element
monitorEvents($0, "mouse"); // logs only mouse events
unmonitor(fn); // stop monitoring
unmonitorEvents($0);

// INSPECTION:
getEventListeners($0); // returns object with all listeners on this element
dir($0); // show object properties in tree view (vs default string)
dirxml($0); // show DOM node in XML view

// TIMING:
console.time("label"); // start timer
console.timeEnd("label"); // stop + print elapsed: "label: 42.3ms"
console.timeLog("label"); // print elapsed without stopping

// GROUPING:
console.group("Label");
console.log("inside group");
console.groupEnd();

console.groupCollapsed("Label"); // starts collapsed

// TABLE:
console.table(data); // formats array of objects as a table
console.table(data, ["id", "name"]); // only show specific columns

// ASSERTION:
console.assert(condition, "message"); // only logs if condition is false
// Like a lightweight throw for debugging: console.assert(arr.length > 0, 'Empty array!')

// COPY:
copy(expression); // copies the result to clipboard
copy(JSON.stringify(complexObject, null, 2));
```

---

## 10. Rendering Panel — Visual Diagnostics

```
More tools → Rendering (or DevTools → ... → More tools → Rendering)

KEY TOGGLES:

Paint Flashing:
  Green overlay = area being repainted this frame
  Ideal: no flashing during CSS animations (transform/opacity only)
  Problem: flashing during scroll = scroll is causing repaints

Layout Shift Regions:
  Blue overlay = Cumulative Layout Shift (CLS) occurring
  Identify which elements are shifting and causing CLS score to increase

Frame Rendering Stats:
  FPS meter overlay on the page (top-right corner)
  GPU memory usage
  Shows in real-time whether you're hitting 60fps

Scrolling Performance Issues:
  Highlights: slow scroll areas (yellow/red)
  Identifies: elements blocking passive scroll (touch handlers on scroll container)

CSS Media Features:
  Force prefers-color-scheme: dark/light (without changing OS setting)
  Force prefers-reduced-motion: reduce (test without OS accessibility setting)
  Force print media query: see print stylesheet

GPU Layer Borders:
  Red borders = GPU compositor layers
  Use to verify `will-change: transform` is creating the expected layer
```

---

## 11. DevTools Shortcuts

```
GLOBAL:
  F12 / Ctrl+Shift+I: Open/close DevTools
  Ctrl+Shift+P: Command palette (run any action by name)
  Ctrl+[: Move to next panel
  Ctrl+]: Move to previous panel
  Ctrl+R: Reload page (while DevTools focused)
  Ctrl+Shift+R: Hard reload (bypass cache)
  Ctrl+L: Clear console

SOURCES (while paused):
  F8:         Resume execution
  F10:        Step over
  F11:        Step into
  Shift+F11:  Step out
  Ctrl+F8:    Deactivate all breakpoints temporarily

NETWORK:
  Ctrl+Shift+E: Clear network log

CONSOLE:
  Up/Down arrows: navigate expression history
  Ctrl+Enter: execute multiline expression
  Shift+Enter: add new line without executing

ELEMENTS:
  Delete: delete selected DOM node
  H: toggle visibility of selected element (adds/removes visibility:hidden)
  Ctrl+Z: undo DOM change (works for last ~5 changes)
```

---

## 12. Good Practices

### ✅ Use conditional breakpoints instead of console.log loops

```javascript
// ❌ Tedious: adding console.log to find which iteration fails
items.forEach((item, i) => {
  console.log(`Item ${i}:`, item); // logs 10,000 times
  processItem(item);
});

// ✅ Conditional breakpoint: right-click line → "Add conditional breakpoint"
// Condition: item.price < 0 || item.id === undefined
// Pauses ONLY on the problematic item — no console spam
```

### ✅ Use logpoints for high-frequency events

```javascript
// ❌ console.log in scroll handler: thousands of logs per session
window.addEventListener("scroll", () => {
  console.log("scroll:", window.scrollY);
});

// ✅ Logpoint in DevTools: add logpoint on the handler line
// "Scroll: {window.scrollY}" — logs to console without pausing, no code change
```

### ✅ "Restart Frame" before reloading to reproduce bugs

```
When paused in a function after a bug:
  Right-click the relevant frame in call stack → "Restart frame"
  Re-examine from the beginning without full page reload
  Faster feedback loop for bugs that are expensive to reproduce
```

---

## 13. Common Debugging Workflows

### Workflow 1 — "Something changed the DOM and I don't know what"

```
1. Elements panel → select the element that's changing
2. Right-click → Break on → Attribute modifications (or Subtree modifications)
3. Trigger the change in the UI
4. Execution pauses with a Sources breakpoint
5. Call stack shows exactly which code modified the DOM
```

### Workflow 2 — "This component re-renders too often"

```
1. React DevTools → Profiler → Record
2. Trigger the interaction
3. Stop recording
4. In the flame chart: find the component → check "Why did this render?"
5. Shows: which prop/state/context changed to trigger the re-render
6. If it's an object prop that "always changes": add React.memo + useMemo on parent
```

### Workflow 3 — "This API request is failing and I don't know why"

```
1. Network panel → XHR/Fetch filter
2. Find the failing request (red status)
3. Click it → Headers tab: see request headers and response headers
4. Response tab: see the full error response body
5. Timing tab: see if it's timing out (long "Waiting" bar)
6. Preview tab: formatted view of error response
7. If redirect: check if "Preserve log" is on and follow the redirect chain
```

### Workflow 4 — "The page is slow/janky"

```
1. Performance → Start recording
2. Interact with the slow part
3. Stop recording
4. Look for: red Long Task bars, dropped FPS frames, wide JS flame chart bars
5. Zoom into the slowest task
6. Find the widest bar → that's the bottleneck
7. Click → "Reveal in Sources" → view the code
8. Consider: useMemo, virtualization, web worker, or algorithmic fix
```

---

## 14. Interview-Level Explanation

> **"How do you use Chrome DevTools to debug a performance problem?"**

**Strong answer:**

> "I approach performance debugging in three phases: measure, identify, fix.
>
> For measuring: I use the Performance panel. I start recording, reproduce the interaction that feels slow, stop recording. The timeline at the top shows FPS — red and yellow areas indicate dropped frames or frame budget overruns. Long Tasks appear as red-hatched bars exceeding 50ms. The flame chart below breaks down what the CPU is actually doing: yellow is JavaScript, purple is style and layout recalculation, green is paint.
>
> For identifying: I zoom into the slowest task by dragging the timeline selector over it. In the flame chart, I look for the widest bar — width represents time. If I see a wide JavaScript bar, I look at its children (below it) to find the actual function responsible. Bottom-up view is useful here — it shows functions sorted by how much 'self time' they consumed, isolating the actual bottleneck rather than callers. If I see many repeated purple 'Recalculate Style' or 'Layout' bars interspersed with JavaScript calls, that's layout thrashing — reading DOM geometry after writing styles.
>
> For memory: Heap Snapshots in the Memory panel. I take a baseline, perform the suspected leaking action several times, force a garbage collection, take a second snapshot, then compare them in the Comparison view. Objects sorted by positive delta that are DOM nodes or component-looking objects are the first suspects. Clicking an instance and looking at its Retainers tree shows exactly which reference path prevents it from being collected.
>
> For React-specific performance, React DevTools Profiler adds a layer: it shows which components re-rendered in a commit, how long each took, and critically 'Why did this render?' — which prop, state, or context changed. That's usually faster than the Performance panel for React-specific questions.
>
> The Rendering panel is often overlooked — enabling Paint Flashing shows green overlays where the browser is repainting each frame. If I see paint flashing during CSS animations, that tells me the animation isn't compositor-only and is triggering main-thread rasterization every frame."
