# 01 — HTTP Protocols

> **"HTTP is the language of the web. Every image, every script, every API call — HTTP. Understanding the protocol isn't academic; it directly determines whether your app loads in 1 second or 8 seconds, whether it degrades gracefully on bad networks, and whether your architecture decisions make the browser work with you or against you."**

HTTP has evolved through three major versions — HTTP/1.1, HTTP/2, and HTTP/3 — each addressing fundamental performance limitations of its predecessor. Understanding how each version works, what bottlenecks it introduces, and how to write code that takes advantage of each protocol's strengths is the difference between a naively implemented frontend and one that delivers excellent performance across all network conditions.

---

## 📚 Table of Contents

1. [HTTP/1.1 — The Foundation and Its Limits](#1-http11--the-foundation-and-its-limits)
2. [HTTP/2 — Multiplexing and Compression](#2-http2--multiplexing-and-compression)
3. [HTTP/3 — QUIC and Zero RTT](#3-http3--quic-and-zero-rtt)
4. [Protocol Negotiation (ALPN)](#4-protocol-negotiation-alpn)
5. [Connection Costing](#5-connection-costing)
6. [Request Prioritization](#6-request-prioritization)
7. [Header Compression (HPACK / QPACK)](#7-header-compression-hpack--qpack)
8. [Server Push (HTTP/2)](#8-server-push-http2)
9. [Resource Hints and Preloading](#9-resource-hints-and-preloading)
10. [Protocol-Aware Frontend Patterns](#10-protocol-aware-frontend-patterns)
11. [Measuring Protocol Performance](#11-measuring-protocol-performance)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. HTTP/1.1 — The Foundation and Its Limits

### The Request-Response Model

```
HTTP/1.1 request:
  GET /api/users HTTP/1.1
  Host: api.example.com
  Accept: application/json
  Authorization: Bearer eyJhbGciOiJ...
  User-Agent: Mozilla/5.0...

Response:
  HTTP/1.1 200 OK
  Content-Type: application/json
  Content-Length: 1247

  {"users": [...]}

CONSTRAINTS:
  1. One request per connection at a time (head-of-line blocking)
  2. Large, repetitive headers (uncompressed, sent in full every request)
  3. Text protocol: verbose, requires parsing
```

### Head-of-Line Blocking

```
HTTP/1.1 pipeline (theoretical):
  Connection: Request 1 → Request 2 → Request 3

  In practice: must wait for each response
  Request 1: 200ms response time → blocks 2 and 3

  [Request 1 ====200ms====][Request 2 ==100ms==][Request 3 =50ms=]
  Total time: 350ms (sequential)

  If Request 2 was fast but Request 1 is slow:
  [Request 1 ====slow============================][Request 2 =fast=]
  Request 2 WAITS even though it's ready
```

### The 6-Connection Workaround

```
Browsers opened up to 6 parallel connections per hostname in HTTP/1.1
to work around head-of-line blocking:

  Connection 1: app.js (400KB) ════════════════════════════
  Connection 2: styles.css (50KB) ═══
  Connection 3: font.woff2 (80KB) ═════
  Connection 4: hero.jpg (200KB) ════════════
  Connection 5: logo.svg (5KB) ═
  Connection 6: (waiting for connection to free up)

  Problem: each connection = TCP handshake (1.5 RTT) + TLS (1-2 RTT)
  6 connections × ~150ms setup = ~900ms overhead
```

### HTTP/1.1 Optimization Tricks (Now Anti-Patterns)

```
DOMAIN SHARDING (avoid in HTTP/2 era):
  Split resources across multiple domains to get more than 6 connections
  img1.example.com, img2.example.com, img3.example.com
  Each domain: 6 connections = 18 parallel connections total
  HTTP/2 made this unnecessary and counterproductive (defeats multiplexing)

SPRITE SHEETS (avoid in HTTP/2 era):
  Combine many images into one large sprite sheet
  Save: N HTTP requests (N - 1 round trips)
  Cost: load entire sprite even if only using 1 icon
  HTTP/2: multiple image requests cost almost nothing (same connection)

FILE BUNDLING (partially obsolete):
  Concatenate many JS files into one large bundle
  Save: N - 1 HTTP round trips
  Cost: all code loaded even if unused
  HTTP/2 + code splitting: many small files loads efficiently
```

---

## 2. HTTP/2 — Multiplexing and Compression

### The Core Innovation — Multiplexing

```
HTTP/2 over ONE TCP connection:

  Single connection (one TCP, one TLS handshake):

  Stream 1 ───[Request]────────────────────[Response 200KB]=====
  Stream 2 ─────────[Request]──[Response 2KB]
  Stream 3 ───────────────[Request]──────────────[Response 50KB]═
  Stream 4 ─────────────────[Request]──[Response 1KB]

  All streams share one connection
  Each has independent flow: no head-of-line blocking at HTTP level
  Browser sends many requests; server interleaves responses

  One TLS handshake for ALL requests (vs 6 for HTTP/1.1)
```

### Binary Framing

```
HTTP/1.1 (text):
  GET /api/users HTTP/1.1\r\n
  Host: example.com\r\n
  \r\n

  Text parsing: find headers by scanning for \r\n sequences
  Text errors are possible (invalid characters, encoding issues)

HTTP/2 (binary frames):
  ┌────────────────────────────────────────────┐
  │ Length (24 bits) │ Type (8) │ Flags (8)   │
  ├────────────────────────────────────────────┤
  │                Stream ID (31 bits)         │
  ├────────────────────────────────────────────┤
  │               Frame Payload               │
  └────────────────────────────────────────────┘

  Frame types: DATA, HEADERS, SETTINGS, WINDOW_UPDATE, RST_STREAM, PUSH_PROMISE...
  Binary: no text parsing, no ambiguity, more efficient
```

### HTTP/2 Features Summary

```
✓ Multiplexing: N requests over 1 connection
✓ Header compression (HPACK): repeated headers sent once, then referenced by index
✓ Server Push: server proactively sends resources (deprecated, replaced by preload)
✓ Stream prioritization: client signals which responses are more important
✓ Flow control: per-stream and per-connection flow control windows
✓ Single TCP connection: one handshake, maintained for all requests

✗ TCP head-of-line blocking still exists:
  HTTP/2 multiplexing is at the HTTP layer
  TCP is still sequential: lost packet blocks ALL streams
  On lossy networks: HTTP/2 can be slower than HTTP/1.1!
```

### HTTP/2 Practical Impact

```javascript
// Feature detection and behavior differences
const connection = performance.getEntriesByType('resource')[0];
console.log(connection.nextHopProtocol); // "h2" for HTTP/2, "http/1.1", "h3"

// HTTP/2 encourages many small files over fewer large ones
// (domain sharding and file concatenation are less important)

// Modern bundle strategy with HTTP/2:
// - Split by route (lazy loading)
// - Split vendor chunks (cache stability)
// - NOT: mega-bundle everything together

// Webpack/Vite code splitting for HTTP/2:
{
  optimization: {
    splitChunks: {
      chunks: 'all',
      // Many smaller chunks: works great over HTTP/2
      // Would be terrible over HTTP/1.1 (many round trips)
      maxSize: 50_000, // 50KB per chunk
    },
  },
}
```

---

## 3. HTTP/3 — QUIC and Zero RTT

### The TCP Problem HTTP/2 Couldn't Solve

```
TCP HEAD-OF-LINE BLOCKING:
  HTTP/2 over TCP: many streams over one TCP connection

  Packet loss scenario:
  ┌─────────────────────────────────────────────────────────────┐
  │ Packet 1 (Stream A, bytes 0-1400)     → received ✓         │
  │ Packet 2 (Stream A, bytes 1400-2800)  → LOST ✗             │
  │ Packet 3 (Stream B, bytes 0-1400)     → received ✓         │
  │ Packet 4 (Stream C, bytes 0-1400)     → received ✓         │
  └─────────────────────────────────────────────────────────────┘

  TCP: cannot deliver Packets 3 and 4 to the app until Packet 2 is retransmitted
  Even though Stream B and Stream C are complete and ready
  TCP's reliability guarantee blocks at the transport layer

  On 1% packet loss: HTTP/2 can perform WORSE than HTTP/1.1 with 6 connections
```

### HTTP/3 and QUIC

```
QUIC (Quick UDP Internet Connections):
  Based on UDP (not TCP)
  Implements reliability at the application layer per-stream
  Packet loss in Stream A: only Stream A waits for retransmission
  Streams B and C: continue unaffected

  TCP HEAD-OF-LINE BLOCKING: ELIMINATED

Additional QUIC features:
  0-RTT resumption: known servers get first request in same round trip as handshake
  Connection migration: connection survives IP address change (WiFi→cellular handoff)
  Built-in TLS 1.3: security baked in, not layered on top
  Faster handshake: 1-RTT (vs TCP 3-way + TLS = 2-3 RTT)
```

### HTTP/3 Connection Setup Comparison

```
HTTP/1.1 (new connection):
  t=0:   SYN (TCP handshake)
  t=1RTT: SYN-ACK
  t=1RTT: ACK + ClientHello (TLS 1.2)
  t=2RTT: ServerHello + Certificate + ServerHelloDone
  t=3RTT: ClientKeyExchange + ChangeCipherSpec
  t=4RTT: HTTP GET
  First byte: 4+ RTTs

HTTP/2 (new connection, TLS 1.3):
  t=0:   SYN
  t=1RTT: SYN-ACK + ClientHello (TLS 1.3)
  t=2RTT: ServerHello + Certificate + Finished
  t=2RTT: HTTP GET (same flight as TLS Finished)
  First byte: 2-3 RTTs

HTTP/3 (new connection, QUIC):
  t=0:   Initial packet (QUIC + TLS 1.3 combined)
  t=1RTT: Handshake complete + first GET in same packet
  First byte: 1-2 RTTs

HTTP/3 (resumed connection, 0-RTT):
  t=0:   Initial packet + Early Data (GET) in SAME packet
  First byte: 0-1 RTTs (can piggyback GET with handshake)
```

### HTTP/3 Performance on Bad Networks

```
Good network (< 0.1% packet loss):
  HTTP/2 and HTTP/3 roughly equivalent
  H2 may be slightly faster (mature optimization)

Bad mobile network (2% packet loss):
  HTTP/3: significant advantage
  - Per-stream loss: only affected stream pauses
  - H2: TCP blocking across all streams

Connection handoff (WiFi to cellular):
  HTTP/2: connection resets, new TCP handshake required
  HTTP/3: connection migration, same connection survives

High-latency connection (300ms RTT, satellite):
  HTTP/3 0-RTT: saves one full round trip
  2 RTTs instead of 3 RTTs = 300ms saved = 10% load time improvement
```

---

## 4. Protocol Negotiation (ALPN)

Browsers and servers negotiate which protocol to use during TLS handshake:

```
TLS ClientHello includes:
  Extension: ALPN (Application-Layer Protocol Negotiation)
  Supported protocols: ["h2", "http/1.1"]

TLS ServerHello includes:
  Selected protocol: "h2"
  (or "h3" via Alt-Svc header for HTTP/3)

HTTP/3 negotiation (different mechanism):
  Server sends Alt-Svc header in HTTP/1.1 or HTTP/2 response:
  Alt-Svc: h3=":443"; ma=86400

  Browser: "this server supports h3 on port 443 (max-age: 86400)"
  Next request: browser tries HTTP/3 (QUIC) on port 443
  Falls back to H2/H1 if UDP is blocked (some firewalls block UDP 443)
```

---

## 5. Connection Costing

Understanding the cost of establishing connections is essential for frontend performance:

```javascript
// Measuring connection overhead with Resource Timing API
const entries = performance.getEntriesByType("resource");

entries.forEach((entry) => {
  const timings = {
    // DNS lookup time
    dns: entry.domainLookupEnd - entry.domainLookupStart,

    // TCP connection time (0 if reused)
    tcp: entry.connectEnd - entry.connectStart,

    // TLS handshake time (subset of TCP time, or 0 if not TLS / reused)
    tls:
      entry.secureConnectionStart > 0
        ? entry.connectEnd - entry.secureConnectionStart
        : 0,

    // Time from request sent to first byte (TTFB)
    ttfb: entry.responseStart - entry.requestStart,

    // Download time
    download: entry.responseEnd - entry.responseStart,

    // Total
    total: entry.responseEnd - entry.startTime,

    // Reused connection? (tcp = 0 means yes)
    reused: entry.connectEnd === entry.connectStart,

    // Protocol
    protocol: entry.nextHopProtocol,
  };

  if (!timings.reused) {
    console.log(
      `New ${timings.protocol} connection: DNS=${timings.dns}ms, TCP=${timings.tcp}ms, TLS=${timings.tls}ms`,
    );
  }
});

// Typical costs (100ms RTT):
// DNS:  20-100ms (first resolution, cached after)
// TCP:  100ms (1 RTT for HTTP/1.1 and HTTP/2)
// TLS:  100-200ms (1-2 RTT for TLS 1.3)
// QUIC: 100ms (1 RTT including TLS)
// 0-RTT: 0ms (reused QUIC session)
```

---

## 6. Request Prioritization

HTTP/2 introduced stream prioritization; HTTP/3 uses a different model (PRIORITY_UPDATE):

```javascript
// Browsers assign priorities automatically based on resource type:
//
// Highest:  Main HTML, Render-blocking CSS
// High:     Preloaded scripts/fonts, XHR/fetch() requests
// Medium:   Scripts loaded late (async/deferred)
// Low:      Images (not in viewport), preload links below fold
// Lowest:   Speculative prefetches

// You can influence prioritization with fetchpriority attribute:
// (supported in Chrome 102+, Firefox 132+, Safari 17.2+)

// Critical LCP image: fetch with high priority
<img src="/hero.jpg" fetchpriority="high" />

// Below-fold image: explicitly low priority
<img src="/footer-image.jpg" fetchpriority="low" loading="lazy" />

// Critical third-party script: increase priority
<script src="https://analytics.example.com/track.js" fetchpriority="high"></script>

// Non-critical prefetch: low priority
<link rel="prefetch" href="/about" fetchpriority="low" />
```

### Fetch Priority via JavaScript

```javascript
// fetchpriority in Fetch API
const criticalData = await fetch("/api/critical", {
  priority: "high", // 'high' | 'low' | 'auto'
});

const nonCritical = await fetch("/api/analytics", {
  priority: "low",
});

// Useful for: lazy-loaded chunks, speculative prefetches
// Don't override auto for most requests — browser knows best
```

---

## 7. Header Compression (HPACK / QPACK)

HTTP/1.1 sends full headers on every request — a significant bandwidth and latency cost:

```
HTTP/1.1 headers (sent in full on EVERY request):
  Accept: */*
  Accept-Encoding: gzip, deflate, br
  Accept-Language: en-US,en;q=0.9
  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiIxMjMifQ.xxx
  Cache-Control: no-cache
  Connection: keep-alive
  Cookie: session=abc123; preferences=dark-mode;lang=en
  Host: api.example.com
  User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...
  Content-Type: application/json

  Total: ~1,500 bytes per request × 100 requests = 150KB of headers alone!
  On 3G: 150KB = ~1.2 seconds just for headers
```

### HPACK (HTTP/2) Header Compression

```
HPACK builds a shared context between client and server:

Static table (predefined common headers):
  Index 2: GET method
  Index 3: POST method
  Index 4: :path = /
  Index 5: :path = /index.html
  ... (61 predefined entries)

Dynamic table (built during session):
  After first request:
    Authorization: Bearer xxx → stored at index 62
    User-Agent: Mozilla/... → stored at index 63
    Cookie: session=... → stored at index 64

  Second request headers:
    ":method: GET" → send index 2 (1 byte vs 15 bytes)
    "Authorization: Bearer xxx" → send index 62 (1 byte vs 400 bytes)

  Compression ratio: often 80-95% on subsequent requests
```

### Impact on Frontend

```javascript
// HTTP/2 header compression makes it practical to:
// 1. Use many authentication tokens without bandwidth penalty
// 2. Send detailed Accept headers (negotiation) for free
// 3. Use verbose feature flags in headers without bloating requests

// Pattern: rich HTTP headers are now efficient
const response = await fetch("/api/data", {
  headers: {
    Authorization: `Bearer ${longToken}`,
    "Accept-Language": navigator.language,
    "X-Feature-Flags": getEnabledFeatures().join(","),
    Accept: "application/json, text/plain, */*",
    // These headers: expensive in HTTP/1.1, compressed to bytes in HTTP/2
  },
});
```

---

## 8. Server Push (HTTP/2)

HTTP/2 Server Push allowed servers to proactively send resources without waiting for a request. It was largely deprecated and removed from major browsers by 2023.

```
SERVER PUSH (deprecated):
  Browser requests: /index.html
  Server sends (without being asked):
    Push promise: /styles.css
    Push promise: /app.js
    HTML response: /index.html

  Browser receives CSS and JS before it parsed the HTML!

WHY IT FAILED:
  - Race conditions: browser might fetch the same resource independently
  - No browser control: server pushes what IT thinks is needed
  - Wasted bandwidth: pushed resources already in browser cache
  - Complex server-side implementation

MODERN REPLACEMENT:
  103 Early Hints: server sends preload hints before full response
  <link rel="preload">: browser-initiated preloading
  These give the browser control — it decides what to fetch
```

### 103 Early Hints (Modern Alternative)

```http
# Server sends 103 before the full response is ready
HTTP/1.1 103 Early Hints
Link: </styles.css>; rel="preload"; as="style"
Link: </app.js>; rel="preload"; as="script"

# ... server processes the request ...

HTTP/1.1 200 OK
Content-Type: text/html
...
```

---

## 9. Resource Hints and Preloading

Resource hints help the browser make better network decisions:

```html
<!-- PRECONNECT: establish TCP+TLS to a domain early -->
<!-- Use for: third-party domains you'll definitely use -->
<link rel="preconnect" href="https://api.example.com" crossorigin />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- DNS-PREFETCH: just DNS lookup (cheaper than preconnect) -->
<!-- Use for: domains you might use, or as fallback for older browsers -->
<link rel="dns-prefetch" href="https://analytics.example.com" />

<!-- PRELOAD: fetch a specific resource with high priority NOW -->
<!-- Use for: critical resources discovered later in the HTML -->
<link rel="preload" href="/fonts/inter.woff2" as="font" crossorigin />
<link rel="preload" href="/hero.jpg" as="image" fetchpriority="high" />
<link rel="preload" href="/critical.css" as="style" />

<!-- PREFETCH: fetch a resource for FUTURE navigation (low priority) -->
<!-- Use for: next page's JS bundle, anticipated user journeys -->
<link rel="prefetch" href="/product.js" as="script" />

<!-- MODULEPRELOAD: preload ES module AND its dependencies -->
<link rel="modulepreload" href="/app.js" />
<!-- Without modulepreload: app.js loads, then discovers imports, then fetches them -->
<!-- With modulepreload: ALL modules fetched in parallel -->
```

### Dynamic Resource Hints

```javascript
// Programmatically add resource hints based on user behavior
function addPreconnect(url) {
  if (!document.querySelector(`link[href="${url}"][rel="preconnect"]`)) {
    const link = document.createElement("link");
    link.rel = "preconnect";
    link.href = url;
    link.crossOrigin = "";
    document.head.appendChild(link);
  }
}

function prefetchPage(href) {
  const link = document.createElement("link");
  link.rel = "prefetch";
  link.href = href;
  document.head.appendChild(link);
}

// On hover: establish connection to API if user will likely navigate
productLinks.forEach((link) => {
  link.addEventListener("mouseenter", () => {
    addPreconnect("https://api.example.com");
    // OR: prefetch the next page
    prefetchPage(link.href);
  });
});
```

---

## 10. Protocol-Aware Frontend Patterns

### Adjust Code Splitting Strategy per Protocol

```javascript
// Vite: code splitting appropriate for HTTP/2
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        // HTTP/2: many small chunks work well (single connection)
        // HTTP/1.1: fewer larger chunks (limited connections)
        manualChunks: {
          // Vendor: stable, long-cached
          vendor: ["react", "react-dom"],
          router: ["react-router-dom"],
          query: ["@tanstack/react-query"],
          // Feature chunks: one per lazy-loaded route
          // These are fetched over the same H2 connection
        },
        // Each route: its own chunk (lazy loaded)
        // Over HTTP/2: 20 small files = same cost as 1 large file
      },
    },
    chunkSizeWarningLimit: 100, // 100KB per chunk is fine over HTTP/2
  },
};
```

### Minimize Round Trips

```javascript
// Each round trip costs 1 RTT (100ms on average connection)
// Minimizing round trips = minimizing perceived latency

// ❌ Sequential requests (3 RTTs):
const user = await fetch("/api/user"); // RTT 1
const orders = await fetch("/api/orders"); // RTT 2 (waits for user)
const products = await fetch("/api/products"); // RTT 3 (waits for orders)

// ✅ Parallel requests (1 RTT):
const [user, orders, products] = await Promise.all([
  fetch("/api/user").then((r) => r.json()),
  fetch("/api/orders").then((r) => r.json()),
  fetch("/api/products").then((r) => r.json()),
]);
// All three send at once, HTTP/2 multiplexes them → 1 RTT for all three

// ✅ Even better: one request that returns all data
const { user, orders, products } = await fetch("/api/dashboard").then((r) =>
  r.json(),
);
// Eliminates the round trips entirely
```

---

## 11. Measuring Protocol Performance

```javascript
// Detect which protocol a resource used
function analyzeProtocols() {
  const resources = performance.getEntriesByType("resource");
  const protocols = {};

  resources.forEach((entry) => {
    const p = entry.nextHopProtocol || "unknown";
    protocols[p] = (protocols[p] || 0) + 1;
  });

  console.table(protocols);
  // Expected output:
  // h3: 45  ← HTTP/3
  // h2: 20  ← HTTP/2
  // http/1.1: 3  ← old third-party
}

// Measure connection reuse rate (should be high with HTTP/2)
function analyzeConnections() {
  const resources = performance.getEntriesByType("resource");

  const stats = resources.reduce(
    (acc, entry) => {
      const newConn = entry.connectEnd > entry.connectStart;
      acc.total++;
      if (newConn) acc.newConnections++;
      return acc;
    },
    { total: 0, newConnections: 0 },
  );

  const reusedPercent = (
    ((stats.total - stats.newConnections) / stats.total) *
    100
  ).toFixed(1);
  console.log(
    `Connection reuse: ${reusedPercent}% (${stats.newConnections} new, ${stats.total} total)`,
  );
  // Good: > 90% reused (one connection serves all requests)
  // Bad: < 70% reused (many new connections = many handshakes)
}

// Network Information API: detect connection quality
const { effectiveType, downlink, rtt, saveData } = navigator.connection ?? {};
console.log(
  `Connection: ${effectiveType}, RTT: ${rtt}ms, Downlink: ${downlink}Mbps`,
);
// effectiveType: 'slow-2g' | '2g' | '3g' | '4g'
// saveData: true if user has Data Saver enabled
```

---

## 12. Good Practices

### ✅ Enable HTTP/2 or HTTP/3 on your server

```nginx
# nginx: HTTP/2
server {
  listen 443 ssl http2;
  # ...
}

# nginx: HTTP/3 (requires nginx 1.25+)
server {
  listen 443 ssl;
  listen 443 quic reuseport;
  http2 on;
  ssl_protocols TLSv1.3;

  add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

### ✅ Preconnect to critical third-party origins

```html
<!-- Always preconnect to origins you'll definitely use -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link rel="preconnect" href="https://api.example.com" crossorigin />
```

### ✅ Use `fetchpriority` for the LCP image

```html
<!-- Hero/LCP image: hint to browser to prioritize this -->
<img src="/hero.jpg" fetchpriority="high" loading="eager" />
```

### ✅ Parallel data fetching

```typescript
// ✅ Fire all requests in parallel — one RTT for multiple responses
const [user, preferences, notifications] = await Promise.all([
  getUser(id),
  getPreferences(id),
  getNotifications(id),
]);
```

---

## 13. Bad Practices

### ❌ Domain sharding in HTTP/2 era

```html
<!-- ❌ HTTP/1.1-era optimization that hurts HTTP/2 -->
<img src="https://img1.example.com/product.jpg" />
<img src="https://img2.example.com/product.jpg" />
<!-- Each subdomain = separate H2 connection = extra handshake = slower! -->

<!-- ✅ Same domain for all assets over HTTP/2 -->
<img src="https://cdn.example.com/product.jpg" />
<!-- One connection serves all images -->
```

### ❌ Unnecessary preconnects

```html
<!-- ❌ Preconnecting to everything wastes connection budget -->
<link rel="preconnect" href="https://every-third-party.example.com" />
<!-- Connection setup costs CPU and network; use for definitely-used origins only -->
<!-- For optional third parties: use dns-prefetch instead -->
```

### ❌ Sequential API requests when parallel is possible

```javascript
// ❌ Waterfall: 3× the latency
const a = await fetchA();
const b = await fetchB();
const c = await fetchC();

// ✅ Parallel
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);
```

---

## 14. Common Mistakes

### Mistake 1 — Assuming HTTP/2 is always better than HTTP/1.1

```
HTTP/2 is slower than HTTP/1.1 in specific scenarios:
  - High packet loss networks (TCP head-of-line blocking across ALL streams)
  - When only 1-2 requests per page (H2 overhead not worth it)
  - Misconfigured server with poor H2 implementation

HTTP/3 was designed to fix the packet loss problem
For most cases: H2 ≥ H1.1, and H3 ≥ H2
Test on realistic network conditions, not just fast WiFi
```

### Mistake 2 — Preloading resources that aren't needed for the current page

```html
<!-- ❌ Preloading everything including what won't be used -->
<link rel="preload" href="/checkout.js" as="script" />
<!-- on homepage! -->
<!-- Wastes bandwidth and competes with resources that ARE needed -->

<!-- ✅ Preload only what's definitely needed for the current page -->
<link rel="preload" href="/hero.webp" as="image" />
<!-- above fold image -->
<link rel="preload" href="/critical.css" as="style" />

<!-- ✅ Prefetch for NEXT page (low priority, happens after current page loads) -->
<link rel="prefetch" href="/checkout.js" as="script" />
```

### Mistake 3 — Not specifying `crossorigin` on preconnect for CORS resources

```html
<!-- ❌ Missing crossorigin: separate connections for CORS vs non-CORS requests -->
<link rel="preconnect" href="https://fonts.gstatic.com" />

<!-- ✅ crossorigin: same connection used for CORS requests -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<!-- Without crossorigin: browser creates a second connection for font CORS request -->
```

---

## 15. Interview-Level Explanation

> **"What are the differences between HTTP/1.1, HTTP/2, and HTTP/3? How do they affect frontend performance?"**

**Strong answer:**

> "HTTP has gone through three major versions, each solving a fundamental bottleneck in its predecessor.
>
> HTTP/1.1 has head-of-line blocking at the request level: only one request can be in flight per connection at a time. Browsers work around this by opening 6 parallel connections per hostname, but each connection requires a TCP handshake and TLS negotiation — 2-3 round trips of overhead. This led to performance tricks like domain sharding, file bundling, and sprite sheets — all hacks to reduce round trips.
>
> HTTP/2 solves request-level blocking with multiplexing: many streams share one TCP connection, each with independent flow control. One handshake serves all requests. Header compression (HPACK) reduces header size by 85-95% on repeated requests. This makes many small files as efficient as few large ones — which is why code splitting became viable with HTTP/2. The catch: TCP still has head-of-line blocking at the packet level. A lost packet blocks ALL streams until retransmission, even streams that have no dependency on the lost packet. On lossy networks, HTTP/2 can actually be slower than HTTP/1.1.
>
> HTTP/3 solves the TCP problem by replacing TCP with QUIC — a UDP-based protocol that implements reliability per-stream. A lost packet only pauses the affected stream, not all streams. QUIC also includes TLS 1.3 natively, enabling 1-RTT connections and 0-RTT resumption for known servers. Connection migration means a mobile user switching from WiFi to cellular keeps their connection alive. On bad networks, HTTP/3 provides substantial performance improvements.
>
> For frontend architecture: HTTP/2 makes domain sharding and excessive bundling antipatterns. Code splitting into many small route-level chunks works well because they all load over one connection. The key optimizations are preconnect for third-party origins, fetchpriority for the LCP image, parallel data fetching, and making sure the server actually speaks HTTP/2 or HTTP/3."

---

## 16. Exercises

### Exercise 1 — Diagnose a waterfall

Looking at a network waterfall, you see these requests in sequence:

- `/api/user` (100ms)
- `/api/orders` (150ms) — starts after `/api/user` completes
- `/api/recommendations` (200ms) — starts after `/api/orders` completes

The page also shows fonts loading 500ms after the HTML, and a hero image loading as the page's LCP at 2.5 seconds.

Identify all problems and write the fixes:

<details>
<summary>Solution</summary>

```
Problems identified:

1. Sequential API requests creating a 450ms waterfall
   /api/user → /api/orders → /api/recommendations
   = 100 + 150 + 200 = 450ms total (should be max(100,150,200) = 200ms)

2. Font loading late (500ms after HTML)
   Font is discovered when CSS is parsed → then connection established → then download

3. LCP image at 2.5 seconds
   Image is discovered late (likely no preload hint)

Fixes:

// 1. Parallelize API requests
const [user, orders, recommendations] = await Promise.all([
  fetch('/api/user').then(r => r.json()),
  fetch('/api/orders').then(r => r.json()),
  fetch('/api/recommendations').then(r => r.json()),
]);
// Time: max(100, 150, 200) = 200ms (vs 450ms)

// 2. Preconnect + preload fonts in <head>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
// Starts font download as soon as HTML is parsed (not after CSS)

// 3. Preload the LCP image
<link rel="preload" href="/hero.jpg" as="image" fetchpriority="high">
// OR
<img src="/hero.jpg" fetchpriority="high" loading="eager" />
// Starts image download immediately, moves LCP from 2.5s to ~0.5s

// Bonus: If orders/recommendations depend on userId from /api/user:
const user = await fetch('/api/user').then(r => r.json());
const [orders, recommendations] = await Promise.all([
  fetch(`/api/orders?userId=${user.id}`).then(r => r.json()),
  fetch(`/api/recommendations?userId=${user.id}`).then(r => r.json()),
]);
// 100ms + max(150, 200) = 300ms (vs 450ms, still better than sequential)
```

</details>

---

## 🔗 Related Topics

- [`networking/02-fetch-and-xhr.md`](./02-fetch-and-xhr.md) — Fetch API built on HTTP
- [`caching/01-http-caching.md`](../caching/01-http-caching.md) — HTTP cache headers
- [`caching/05-cdn-strategies.md`](../caching/05-cdn-strategies.md) — CDN over HTTP/2 and HTTP/3
- [`browser-internals/08-critical-rendering-path.md`](../browser-internals/08-critical-rendering-path.md) — Network requests in CRP
- [`performance/08-bundle-optimization.md`](../performance/08-bundle-optimization.md) — Bundle strategy for HTTP/2

---

<div align="center">

**Next:** [`networking/02-fetch-and-xhr.md`](./02-fetch-and-xhr.md) →

</div>
