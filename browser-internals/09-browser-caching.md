# 09 — Browser Caching

> **"The fastest network request is the one you never make. Browser caching is the only optimization that can eliminate the critical rendering path entirely on repeat visits — turning a 2-second load into a 50-millisecond one."**

Browser caching is one of the highest-leverage performance optimizations available. Get it right and repeat visitors experience near-instant page loads. Get it wrong and users either receive stale content or bypass the cache entirely, paying full network cost every time. This document covers the complete browser caching model: HTTP cache semantics, cache control headers, Service Worker caching, memory cache, disk cache, and the practical strategies that make caching reliable and safe.

---

## 📚 Table of Contents

1. [The Browser Cache Hierarchy](#1-the-browser-cache-hierarchy)
2. [HTTP Caching — The Foundation](#2-http-caching--the-foundation)
3. [Cache-Control Header In Depth](#3-cache-control-header-in-depth)
4. [ETag and Last-Modified — Validation](#4-etag-and-last-modified--validation)
5. [Cache Busting Strategies](#5-cache-busting-strategies)
6. [Vary Header](#6-vary-header)
7. [Service Worker Cache vs HTTP Cache](#7-service-worker-cache-vs-http-cache)
8. [Memory Cache vs Disk Cache](#8-memory-cache-vs-disk-cache)
9. [Push Cache (HTTP/2)](#9-push-cache-http2)
10. [Caching Strategies by Resource Type](#10-caching-strategies-by-resource-type)
11. [Cache Invalidation — The Hard Problem](#11-cache-invalidation--the-hard-problem)
12. [IndexedDB for Application-Level Caching](#12-indexeddb-for-application-level-caching)
13. [Measuring Cache Performance](#13-measuring-cache-performance)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. The Browser Cache Hierarchy

The browser has multiple caching layers, each with different characteristics:

```
Request for a resource:

1. SERVICE WORKER CACHE
   (if SW is registered and handles the fetch event)
   → Can return any response, bypass network entirely
   → Programmable — you control what's cached and for how long
   → Persists across browser sessions
   └── If SW doesn't handle: ↓

2. MEMORY CACHE (RAM)
   → Fastest — sub-millisecond access
   → Only for current browsing session (cleared on tab close)
   → Small capacity (hundreds of MB)
   → Duplicate requests within a page use memory cache
   └── If not in memory cache: ↓

3. DISK CACHE (HTTP Cache)
   → HTTP cache semantics (Cache-Control, ETag, etc.)
   → Persists across sessions (until max-age expires or evicted)
   → Larger capacity (gigabytes)
   → Validated against server using conditional requests
   └── If not in disk cache or expired: ↓

4. PUSH CACHE (HTTP/2)
   → Short-lived cache for server-pushed resources
   → Per-session, very short TTL
   └── If not in push cache: ↓

5. NETWORK REQUEST
   → Full HTTP request to server
   → Response cached according to Cache-Control headers
```

---

## 2. HTTP Caching — The Foundation

HTTP caching is controlled by response headers sent by the server. When a resource is cached, future requests can be served directly from the browser's disk cache without a network round-trip.

### The Cache Decision Flow

```
Request received by browser:

1. Is there a cached response?
   NO → fetch from network (fresh miss)

2. Is the cached response fresh? (max-age not exceeded)
   YES → serve from cache (cache hit — no network)
   NO → go to step 3

3. Does the cache have validation info? (ETag or Last-Modified)
   NO → fetch from network (stale, no validator)
   YES → conditional request to server:
     - If unchanged: server returns 304 Not Modified
       → serve from cache (revalidated — cheap network)
     - If changed: server returns 200 with new content
       → serve new content, update cache (cache miss)
```

### Cache Response Headers

```http
HTTP/1.1 200 OK
Content-Type: text/css
Content-Length: 45823
Last-Modified: Mon, 01 Jan 2024 12:00:00 GMT
ETag: "abc123def456"
Cache-Control: public, max-age=31536000, immutable
Date: Mon, 15 Jan 2024 10:30:00 GMT
```

---

## 3. Cache-Control Header In Depth

`Cache-Control` is the primary mechanism for controlling HTTP caching behavior.

### Directives Reference

#### Freshness Directives

```http
Cache-Control: max-age=3600
# Resource is fresh for 3600 seconds (1 hour) from response time
# No network request during this time

Cache-Control: s-maxage=3600
# Like max-age but for shared caches (CDNs, proxies) only
# Browser uses max-age; CDN uses s-maxage

Cache-Control: max-stale=600
# Client willing to accept stale response up to 600 seconds past expiry
# (request directive — sent by browser, usually not manually set)

Cache-Control: stale-while-revalidate=600
# Serve stale while revalidating in background for up to 600s
# After max-age expires, user gets stale response immediately
# Browser revalidates in background — next request gets fresh

Cache-Control: stale-if-error=86400
# If revalidation fails (server unreachable), serve stale for 24h
# Useful for resilience: degraded experience > no experience
```

#### Cacheability Directives

```http
Cache-Control: public
# May be cached by browser AND shared caches (CDN, proxy)
# Use for: publicly accessible resources (CSS, JS, images)

Cache-Control: private
# May ONLY be cached by browser (not CDN/proxy)
# Use for: user-specific content (dashboard data, personalized pages)

Cache-Control: no-store
# NEVER cache — don't even write to disk
# Use for: sensitive data (banking, auth tokens, PII)
# Each request always hits the server

Cache-Control: no-cache
# CAN cache — but MUST revalidate before serving
# Confusing name! "no-cache" doesn't mean "don't cache"
# It means "don't serve without checking freshness"
# Use for: content that changes often and must always be current
```

#### Other Directives

```http
Cache-Control: immutable
# Content will NEVER change while max-age is valid
# Browser won't send conditional revalidation requests
# Use for: content-hashed assets (/app.abc123.js)
# Saves a network round-trip on reload (very valuable)

Cache-Control: must-revalidate
# Once stale, MUST revalidate — cannot serve stale even if network fails
# Use with: max-age for strict freshness requirements

Cache-Control: proxy-revalidate
# Like must-revalidate but for proxies/CDNs only

Cache-Control: no-transform
# CDN/proxy must not modify the content
# Use for: images that would be compressed/resized by CDN (not desired)
```

### Combining Directives

```http
# Long-cached immutable asset (content-hashed URL)
Cache-Control: public, max-age=31536000, immutable

# API response: cache privately, check freshness on every request
Cache-Control: private, no-cache

# HTML page: short cache with stale-while-revalidate
Cache-Control: public, max-age=300, stale-while-revalidate=86400

# Sensitive user data: never cache
Cache-Control: no-store, no-cache

# CDN-optimized: cached at CDN for 1 day, browser for 1 hour
Cache-Control: public, max-age=3600, s-maxage=86400
```

---

## 4. ETag and Last-Modified — Validation

When a cached resource expires, the browser can check if the content has changed using **conditional requests** rather than downloading the full resource again.

### ETag (Entity Tag)

```http
# Server response:
ETag: "d41d8cd98f00b204e9800998ecf8427e"
# ETag is a unique identifier for the resource version
# Can be any string: hash of content, version number, timestamp hash

# Browser stores the ETag.
# When cache expires, browser sends conditional request:
GET /styles.css HTTP/1.1
If-None-Match: "d41d8cd98f00b204e9800998ecf8427e"

# If content unchanged:
HTTP/1.1 304 Not Modified
# Browser uses cached version — saves downloading the full 80KB
# Only headers transferred (~200 bytes)

# If content changed:
HTTP/1.1 200 OK
ETag: "newetag12345"
[full response body]
```

### Last-Modified

```http
# Server response:
Last-Modified: Wed, 01 Jan 2025 12:00:00 GMT

# Browser sends conditional request:
GET /image.jpg HTTP/1.1
If-Modified-Since: Wed, 01 Jan 2025 12:00:00 GMT

# If not modified:
HTTP/1.1 304 Not Modified
# If modified:
HTTP/1.1 200 OK
[full response body]
```

### ETag vs Last-Modified

| Feature        | ETag                            | Last-Modified                  |
| -------------- | ------------------------------- | ------------------------------ |
| Precision      | Exact content hash              | 1-second granularity           |
| Computation    | Server must compute hash        | Already available (file mtime) |
| Reliability    | Perfect — any change detected   | Sub-second changes missed      |
| Bandwidth      | Same (both just headers on 304) | Same                           |
| Recommendation | Preferred when supported        | Fallback                       |

### Strong vs Weak ETags

```http
# Strong ETag: byte-for-byte identical
ETag: "abc123"

# Weak ETag: semantically equivalent (acceptable for range requests)
ETag: W/"abc123"

# If-None-Match only matches strong ETags by default
# Use strong ETags for most cases
```

---

## 5. Cache Busting Strategies

Long-lived caches are powerful but create a problem: how do you update resources when they change?

### Strategy 1 — Content Hashing (Recommended)

```
Include a hash of the file contents in the filename or URL.
When content changes, hash changes → URL changes → new download.
Old URL cached forever (immutable).

Example:
  styles.abc123de.css  ← production hash, cached for 1 year

  File changes → build system generates new hash:
  styles.xyz789ab.css  ← new file, new cache entry

  HTML references the new URL → browser downloads the new file
  Old URL still cached (but never referenced) → eventually evicted

Benefits:
  - Assets cached forever (immutable) — maximum caching benefit
  - Updates are instant (new URL = new download)
  - No cache invalidation needed — new URL = new resource

Headers:
  Cache-Control: public, max-age=31536000, immutable
```

### Strategy 2 — Query String Versioning

```
Append a version/timestamp as query parameter.
Less reliable than path-based hashing (some CDNs ignore query strings).

Example:
  /styles.css?v=2.1.0   or   /styles.css?t=1704067200

  Headers:
  Cache-Control: public, max-age=86400

  Less ideal: CDN may normalize/strip query strings
             Cache key depends on query string support
```

### Strategy 3 — Path Versioning

```
Include version in path:
  /v2/styles.css   or   /2024-01-15/app.js

  Good for: API versioning (/api/v2/users)
  Less common for assets
```

### Build Tool Integration

```javascript
// Webpack: [contenthash] in output filename
output: {
  filename: '[name].[contenthash:8].js',
  chunkFilename: '[name].[contenthash:8].chunk.js',
}

// Vite: uses [hash] by default
build: {
  rollupOptions: {
    output: {
      entryFileNames: '[name]-[hash].js',
      assetFileNames: '[name]-[hash].[ext]',
    }
  }
}
```

---

## 6. Vary Header

The `Vary` header tells caches that the response varies based on certain request headers. It's critical for serving different content to different clients.

### How Vary Works

```http
# Server response:
Vary: Accept-Encoding
# "My response varies based on what encoding the client accepts"
# Cache must store SEPARATE cached versions for:
#   - Accept-Encoding: gzip   → gzipped response
#   - Accept-Encoding: br     → Brotli response
#   - Accept-Encoding: (none) → uncompressed response
```

### Common Vary Use Cases

```http
# Content negotiation: different format for different clients
Vary: Accept
# Stores separate cache entry for:
#   Accept: text/html    → HTML response
#   Accept: application/json → JSON response

# Compression negotiation
Vary: Accept-Encoding
# Most common — almost all servers use this

# Language negotiation
Vary: Accept-Language
# Different cached response for en-US vs fr-FR vs ja-JP

# Multiple headers
Vary: Accept-Encoding, Accept-Language
# Cache must match BOTH headers — can create many cache entries

# User-specific content (don't cache at all if truly personal)
Vary: Cookie  # or Authorization
# This effectively disables CDN caching
# Each unique cookie value = separate cache entry = cache miss on CDN
# AVOID: use Cache-Control: private instead for user-specific content
```

### Vary: \* — Never Cache

```http
Vary: *
# Means: response varies on EVERYTHING
# Effectively uncacheable — every request must go to origin
# Use only for: truly dynamic responses that can't be cached
```

---

## 7. Service Worker Cache vs HTTP Cache

These are distinct caching systems that can interact — and sometimes conflict.

### The Difference

```
HTTP Cache (browser disk cache):
  - Controlled by HTTP response headers
  - Automatic — no code to write
  - Browser manages lifecycle
  - Can be bypassed with hard refresh (Ctrl+Shift+R)
  - No API to programmatically control

Service Worker Cache (Cache API):
  - Controlled by JavaScript in the Service Worker
  - Manual — you write the code
  - You manage what's cached and when it expires
  - Intercepts fetch events before HTTP cache
  - Full programmatic control
```

### The Interaction

```
fetch() request:
  ↓
Service Worker fetch event handler (if registered)
  ↓ (if SW calls fetch() / passes through)
HTTP Cache check
  ↓ (if not in HTTP cache or expired)
Network request
  ↓ response
HTTP Cache stores response (per Cache-Control)
  ↓
Service Worker receives response
  ↓ (if SW caches it)
Cache API stores response
  ↓
Page receives response
```

### SW Cache Bypassing HTTP Cache

```javascript
// In Service Worker: force bypass of HTTP cache
self.addEventListener("fetch", (event) => {
  event.respondWith(
    // cache: 'no-store' bypasses the HTTP cache
    fetch(event.request, { cache: "no-store" }).then((response) => {
      // Cache in SW cache (Cache API) with our own TTL logic
      cacheWithTTL(event.request, response.clone());
      return response;
    }),
  );
});
```

### `updateViaCache` for Service Worker Script

```javascript
// Ensure SW script itself is always fresh
navigator.serviceWorker.register("/sw.js", {
  updateViaCache: "none", // bypass HTTP cache for sw.js
  // Browser checks for new SW version on every navigation
  // Without this: old SW might be served from cache
});
```

---

## 8. Memory Cache vs Disk Cache

### Memory Cache

```
Memory cache characteristics:
  - Stores resources in RAM during the current browser session
  - Access time: sub-millisecond (memory read)
  - Capacity: limited (a few hundred MB max)
  - Lifetime: tab session only (cleared when tab closes)
  - Primary use: duplicate requests within a single page

When memory cache is used:
  Same resource requested multiple times on same page:
    <img src="logo.png"> appears in both header AND footer
    Second request served from memory — zero latency

Memory cache ignores Cache-Control no-store:
  Resources requested multiple times in the same page load
  are served from memory even if they have no-store
  (this is browser-specific behavior, not spec-required)
```

### Disk Cache (HTTP Cache)

```
Disk cache characteristics:
  - Stores resources on disk (persistent storage)
  - Access time: ~1-10ms (disk read)
  - Capacity: gigabytes (browser-managed)
  - Lifetime: until max-age expires or browser evicts
  - Survives tab and browser restarts

Disk cache respects Cache-Control fully:
  max-age: how long to store
  no-store: don't store at all
  no-cache: store but revalidate before serving
```

### Checking Which Cache Was Used

```javascript
// PerformanceResourceTiming.transferSize: 0 = served from cache
const resources = performance.getEntriesByType("resource");
resources.forEach((entry) => {
  if (entry.transferSize === 0 && entry.decodedBodySize > 0) {
    console.log("From cache:", entry.name);
  } else if (entry.transferSize > 0) {
    console.log(
      "From network:",
      entry.name,
      entry.transferSize + " bytes transferred",
    );
  }
});
```

---

## 9. Push Cache (HTTP/2)

HTTP/2 Server Push allowed servers to proactively send resources before the browser asked. The Push Cache held these pushed resources briefly.

### Current Status

```
HTTP/2 Server Push status (2024):
  - Chrome removed Server Push support (Chrome 106, September 2022)
  - Firefox: limited support
  - Safari: limited support
  - General consensus: preload hints are better than Server Push

  Push Cache is effectively deprecated.
  Use <link rel="preload"> instead.
```

### Why Preload Won

```
Server Push problems:
  - Server pushes resources that browser might already have cached
  - Wasted bandwidth pushing already-cached assets
  - Push can't check browser cache before pushing
  - Priority conflicts with browser's own resource loading

<link rel="preload"> advantages:
  - Browser decides whether to fetch (checks cache first)
  - No wasted bandwidth for cached resources
  - Works with HTTP/1.1 as well as HTTP/2
  - Better priority control via fetchpriority
```

---

## 10. Caching Strategies by Resource Type

Different resources have different freshness requirements and update frequencies. Use the right strategy for each.

### HTML Pages

```http
# HTML changes on every deploy — validate on every request
Cache-Control: no-cache, must-revalidate
# OR: short max-age with stale-while-revalidate
Cache-Control: public, max-age=300, stale-while-revalidate=86400

# Why no-cache for HTML:
# HTML references versioned assets (app.abc123.js)
# If HTML is stale: references old asset URLs → old content served
# HTML must always be fresh — it's the anchor of the versioning system
```

### JavaScript and CSS (Content-Hashed)

```http
# Long-lived immutable cache — URL changes when content changes
Cache-Control: public, max-age=31536000, immutable
# 1 year: URL contains content hash, will never be stale
# immutable: browser won't revalidate during max-age (saves RTT on reload)
```

### Images

```http
# Most images: moderate cache
Cache-Control: public, max-age=86400
# 24 hours — reasonable freshness for most images

# Static brand assets (logo, etc.): long cache
Cache-Control: public, max-age=2592000
# 30 days

# User-uploaded images with changing content:
Cache-Control: public, max-age=3600
# 1 hour — balance freshness vs network

# Content-addressed images (hash in URL):
Cache-Control: public, max-age=31536000, immutable
```

### Fonts

```http
# Fonts rarely change — long cache + CORS
Cache-Control: public, max-age=31536000, immutable

# Fonts require CORS for cross-origin:
Access-Control-Allow-Origin: *
# Without this: browser won't cache and reuse cross-origin fonts
```

### API Responses

```http
# Public API data: moderate cache + stale-while-revalidate
Cache-Control: public, max-age=60, stale-while-revalidate=600
# Fresh for 1 minute, stale-while-revalidate for 10 minutes after

# User-specific API data: private, short cache
Cache-Control: private, max-age=30
# Only in browser cache, 30 seconds

# Sensitive data: no cache
Cache-Control: no-store

# Real-time data: no cache
Cache-Control: no-store, no-cache
```

### Summary Table

| Resource        | Cache-Control                                    | Notes                    |
| --------------- | ------------------------------------------------ | ------------------------ |
| HTML            | `no-cache` or short max-age                      | Must stay fresh          |
| JS/CSS (hashed) | `public, max-age=31536000, immutable`            | URL changes with content |
| Images (static) | `public, max-age=86400`                          | 24 hours                 |
| Fonts           | `public, max-age=31536000, immutable`            | Rarely change            |
| API (public)    | `public, max-age=60, stale-while-revalidate=600` | Balanced freshness       |
| API (private)   | `private, max-age=30`                            | User-specific            |
| API (sensitive) | `no-store`                                       | Never cache              |
| Service Worker  | `no-cache` (via updateViaCache)                  | Always check for updates |

---

## 11. Cache Invalidation — The Hard Problem

> _"There are only two hard things in Computer Science: cache invalidation and naming things."_ — Phil Karlton

Cache invalidation is the process of removing or updating cached content before it expires. It's hard because HTTP caches are distributed — you can't directly control what browsers have cached.

### The Fundamental Problem

```
You deploy new CSS:
  /styles.css → updated with new design

But users have:
  /styles.css cached with max-age=86400 (1 day)

For up to 24 hours after your deploy:
  → Users see old design
  → You can't reach into their cache and delete /styles.css
  → No server-side mechanism to force browser cache refresh
```

### Solution 1 — Content Hashing (Prevents the Problem)

```
Never cache mutable URLs.
Give immutable URLs to immutable content.
Change the URL when content changes.

/styles.css         → AVOID (mutable URL)
/styles.abc123.css  → USE (immutable URL, cached forever)
```

### Solution 2 — Short max-age + stale-while-revalidate

```http
# CSS with known deploy cycle:
Cache-Control: public, max-age=300, stale-while-revalidate=86400

# Fresh for 5 minutes
# After 5 minutes: stale (but served immediately while revalidating)
# After 24 hours: stale, require network

# User gets stale for max 5 minutes after deploy
# Performance: stale is served immediately (no waiting for network)
# Freshness: guaranteed within 5 minutes + 1 revalidation
```

### Solution 3 — Purge CDN Cache on Deploy

```bash
# After deploying new HTML (invalidate CDN cache programmatically):

# Cloudflare:
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  --data '{"files": ["https://example.com/", "https://example.com/index.html"]}'

# AWS CloudFront:
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"  # or specific paths
```

### Service Worker Cache Invalidation

```javascript
// Invalidate SW cache on deploy: version the cache name
const CACHE_VERSION = "v2.1.0"; // bump this on every deploy

self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) =>
      Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_VERSION)
          .map((name) => caches.delete(name)), // delete old caches
      ),
    ),
  );
});
```

---

## 12. IndexedDB for Application-Level Caching

For application data (not HTTP resources), IndexedDB provides a programmable persistent storage layer with more control than the Cache API.

### When to Use IndexedDB Caching

```
Use IndexedDB for:
  - API response data with complex TTL logic
  - Large datasets that need offline access
  - User-generated content before sync
  - Application state persistence
  - Search indexes
  - Large binary data (images, PDFs)

Use Cache API for:
  - HTTP resources (HTML, CSS, JS, images)
  - Response objects with headers
  - Service Worker caching

Use localStorage/sessionStorage for:
  - Small key-value data (< 5MB)
  - Simple settings/preferences
  - Session data
```

### IndexedDB Cache Pattern

```javascript
class IDBCache {
  constructor(dbName = "app-cache", storeName = "responses", version = 1) {
    this._dbName = dbName;
    this._storeName = storeName;
    this._db = null;
    this._ready = this._init(version);
  }

  async _init(version) {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this._dbName, version);

      request.onupgradeneeded = (event) => {
        const db = event.target.result;
        const store = db.createObjectStore(this._storeName, { keyPath: "key" });
        store.createIndex("expiresAt", "expiresAt", { unique: false });
      };

      request.onsuccess = (event) => {
        this._db = event.target.result;
        resolve(this._db);
      };

      request.onerror = () => reject(request.error);
    });
  }

  async _getStore(mode = "readonly") {
    const db = await this._ready;
    return db.transaction([this._storeName], mode).objectStore(this._storeName);
  }

  async get(key) {
    const store = await this._getStore();
    const entry = await new Promise((resolve, reject) => {
      const req = store.get(key);
      req.onsuccess = () => resolve(req.result);
      req.onerror = () => reject(req.error);
    });

    if (!entry) return null;
    if (entry.expiresAt && Date.now() > entry.expiresAt) {
      await this.delete(key); // expired — remove and return null
      return null;
    }

    return entry.value;
  }

  async set(key, value, ttlSeconds = 300) {
    const store = await this._getStore("readwrite");
    return new Promise((resolve, reject) => {
      const req = store.put({
        key,
        value,
        expiresAt: ttlSeconds ? Date.now() + ttlSeconds * 1000 : null,
        cachedAt: Date.now(),
      });
      req.onsuccess = () => resolve();
      req.onerror = () => reject(req.error);
    });
  }

  async delete(key) {
    const store = await this._getStore("readwrite");
    return new Promise((resolve, reject) => {
      const req = store.delete(key);
      req.onsuccess = () => resolve();
      req.onerror = () => reject(req.error);
    });
  }

  async purgeExpired() {
    const store = await this._getStore("readwrite");
    const now = Date.now();
    const index = store.index("expiresAt");

    return new Promise((resolve, reject) => {
      // Get all entries with expiresAt <= now
      const range = IDBKeyRange.upperBound(now);
      const request = index.openCursor(range);
      let deleted = 0;

      request.onsuccess = (event) => {
        const cursor = event.target.result;
        if (cursor) {
          cursor.delete();
          deleted++;
          cursor.continue();
        } else {
          resolve(deleted);
        }
      };

      request.onerror = () => reject(request.error);
    });
  }
}

// Usage
const cache = new IDBCache();

// Cache an API response for 5 minutes
const data = await cache.get("users:list");
if (!data) {
  const fresh = await fetch("/api/users").then((r) => r.json());
  await cache.set("users:list", fresh, 300); // 5 min TTL
  return fresh;
}
return data;
```

---

## 13. Measuring Cache Performance

### Cache Hit Rate

```javascript
// Measure cache effectiveness
class CacheMetrics {
  constructor() {
    this._hits = 0;
    this._misses = 0;
    this._track();
  }

  _track() {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType !== "resource") continue;

        if (entry.transferSize === 0 && entry.decodedBodySize > 0) {
          this._hits++;
        } else if (entry.transferSize > 0) {
          this._misses++;
        }
      }
    });
    observer.observe({ entryTypes: ["resource"] });
  }

  get hitRate() {
    const total = this._hits + this._misses;
    return total > 0 ? ((this._hits / total) * 100).toFixed(1) + "%" : "N/A";
  }

  report() {
    console.log(`Cache hit rate: ${this.hitRate}`);
    console.log(`Hits: ${this._hits}, Misses: ${this._misses}`);
  }
}

const metrics = new CacheMetrics();
window.addEventListener("load", () => setTimeout(() => metrics.report(), 1000));
```

### Bytes Saved by Cache

```javascript
// Estimate bandwidth saved by caching
const resources = performance.getEntriesByType("resource");
let bytesSaved = 0;

resources.forEach((entry) => {
  if (entry.transferSize === 0 && entry.decodedBodySize > 0) {
    bytesSaved += entry.decodedBodySize;
  }
});

console.log(
  "Bytes saved by cache:",
  (bytesSaved / 1024 / 1024).toFixed(2) + "MB",
);
```

### Checking Cache Headers in DevTools

```
DevTools → Network tab → click any resource → Headers tab

Response Headers:
  Cache-Control: public, max-age=31536000, immutable
  ETag: "abc123"
  Age: 3600  ← seconds since cached at CDN/proxy (not browser)

Request Headers (on subsequent requests):
  If-None-Match: "abc123"  ← conditional request
  If-Modified-Since: ...    ← conditional request

Status codes:
  200 (from cache) — served from disk/memory cache
  200 (from network) — downloaded fresh
  304 Not Modified — validated with server, using cache
```

---

## 14. Good Practices

### ✅ Use content hashing for all static assets

```javascript
// Build tool configuration (Webpack, Vite)
// All JS, CSS, images get content-hash in filename
// Enables immutable caching
```

### ✅ Set `no-cache` for HTML with `immutable` for hashed assets

```nginx
# Nginx configuration
location ~* \.(js|css|png|jpg|webp|woff2)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}

location ~* \.html$ {
  add_header Cache-Control "no-cache, must-revalidate";
  add_header ETag $request_id; # unique per response
}
```

### ✅ Use `stale-while-revalidate` for API responses

```http
# Balance freshness and performance
Cache-Control: public, max-age=60, stale-while-revalidate=600
# User always gets fast response (from cache)
# Cache updated in background every 60s
```

### ✅ Purge CDN cache on deployment

```bash
# Part of your CI/CD pipeline:
# Deploy new assets → purge HTML from CDN
# New HTML references new hashed asset URLs → users get new version
```

### ✅ Set proper `Vary` headers

```http
# Always set Vary: Accept-Encoding when using compression
Vary: Accept-Encoding

# Avoid Vary: Cookie unless necessary (kills CDN caching)
```

---

## 15. Bad Practices

### ❌ Long-caching mutable URLs

```http
# ❌ Main CSS with long cache but no URL versioning
GET /styles.css
Cache-Control: public, max-age=86400

# After deploy: users see old styles for up to 24 hours
# No way to invalidate browser-side cache
```

### ❌ `no-cache` for hashed assets

```http
# ❌ Wasteful: content-hashed URL doesn't need revalidation
GET /app.abc123.js
Cache-Control: no-cache
# Forces conditional request on every load
# Defeats the purpose of content hashing
```

### ❌ Missing `immutable` on versioned assets

```http
# ❌ Missing immutable: browser sends conditional request on reload
Cache-Control: public, max-age=31536000
# On Ctrl+R (reload): browser sends If-None-Match anyway
# Server responds 304 — unnecessary round-trip

# ✅ With immutable: browser skips revalidation during max-age
Cache-Control: public, max-age=31536000, immutable
```

### ❌ Caching user-specific data at CDN level

```http
# ❌ Private data cached at CDN — served to wrong user
Cache-Control: public, max-age=300  # on user profile endpoint
# Attacker could receive another user's profile from CDN cache!

# ✅ Private data for browser cache only
Cache-Control: private, max-age=300
```

### ❌ `no-store` for resources that should be cached

```http
# ❌ Overly cautious: no-store for public static assets
Cache-Control: no-store

# Forces full download on every visit
# Every user, every page load, pays full network cost
```

---

## 16. Common Mistakes

### Mistake 1 — Confusing `no-cache` and `no-store`

```
no-cache:
  "You CAN cache this, but MUST revalidate before serving"
  Resource IS stored on disk
  Conditional request sent on each use
  If server says 304: cached version served

no-store:
  "Do NOT store this anywhere"
  Resource is NOT stored to disk
  Every request is a fresh download
  No revalidation — full download every time
```

### Mistake 2 — Forgetting `immutable` on content-hashed assets

```
Without immutable:
  User presses F5 (reload) → browser sends If-None-Match → server 304 → user gets cached content
  → Extra round-trip that could be avoided

With immutable:
  User presses F5 → browser skips revalidation → serves from cache immediately
  → Zero network request
```

### Mistake 3 — Cache-Control without ETag/Last-Modified

```
If max-age expires and there's no ETag/Last-Modified:
  Browser can't send conditional request (no validator)
  Must download full resource again even if unchanged

Always set ETags (or Last-Modified) for potentially revalidatable resources
```

### Mistake 4 — Not testing cache behavior

```javascript
// Many developers never test their actual caching behavior
// Tools to verify:

// 1. DevTools → Network → reload → check "from cache" vs "from disk cache"
// 2. Lighthouse → "Serve static assets with an efficient cache policy"
// 3. WebPageTest: repeat view test (second visit experience)
// 4. Response headers inspection in DevTools
```

---

## 17. Interview-Level Explanation

> **"How does browser caching work? What's the difference between `no-cache` and `no-store`? How do you handle cache invalidation?"**

**Strong answer:**

> "Browser caching has a hierarchy: Service Worker cache (if registered), then memory cache (RAM, current session only), then disk cache (HTTP cache, persistent). Most caching discussion is about the HTTP disk cache.
>
> The HTTP cache is controlled by the `Cache-Control` response header. `max-age=31536000` means the browser considers the resource fresh for one year and won't make a network request at all. Once expired, the browser sends a conditional request using the stored ETag (`If-None-Match`) or Last-Modified date (`If-Modified-Since`). If unchanged, the server returns 304 Not Modified — just headers, no body — and the browser uses the cached version. This saves bandwidth while guaranteeing freshness.
>
> `no-cache` is widely misunderstood — it doesn't mean 'don't cache.' It means 'cache this but always revalidate before serving.' The resource is stored on disk, but every use sends a conditional request. `no-store` means 'never store this at all' — full download every request. Use `no-store` for sensitive data like banking transactions, `no-cache` for HTML that must always be current but benefits from conditional requests.
>
> Cache invalidation is solved architecturally, not mechanically. You can't force browsers to clear their caches. The solution is to never cache mutable URLs — instead, include a content hash in the filename (like `app.abc123.js`) and set `Cache-Control: immutable, max-age=31536000`. When content changes, the hash changes, the URL changes, and browsers automatically download the new version. HTML should use `no-cache` so it's always revalidated — and since HTML references the new hashed asset URLs, users get updated content on next visit. The `immutable` directive additionally prevents browsers from sending conditional requests on reload, saving an extra round-trip for assets during their cache lifetime."

---

## 18. Exercises

### Exercise 1 — Choose the right Cache-Control

For each resource, choose the optimal `Cache-Control` header and explain why:

```
a) /index.html — changes on every deploy
b) /app.a1b2c3d4.js — content-hashed JavaScript bundle
c) /api/user/profile — user-specific JSON data, sensitive
d) /api/products — public product catalog, updates every 5 minutes
e) /logo.svg — never changes
f) /auth/token — authentication token response
g) /images/hero.jpg — changes with seasonal campaigns (~monthly)
```

<details>
<summary>Answers</summary>

```
a) /index.html:
   Cache-Control: no-cache, must-revalidate
   Reason: HTML must always be current (references hashed assets).
   Revalidation with ETag is fast (304 response).

b) /app.a1b2c3d4.js:
   Cache-Control: public, max-age=31536000, immutable
   Reason: Content hash in URL guarantees uniqueness.
   Can be cached forever — URL changes when content changes.
   `immutable` prevents conditional requests on reload.

c) /api/user/profile:
   Cache-Control: private, no-cache
   Reason: User-specific, must always be fresh.
   `private` = browser cache only (not CDN).
   `no-cache` = always revalidate (ETag conditional request).

d) /api/products:
   Cache-Control: public, max-age=300, stale-while-revalidate=600
   Reason: Public data, 5-minute update cycle.
   `max-age=300` = fresh for 5 minutes.
   `stale-while-revalidate=600` = serve stale immediately
   while revalidating in background for up to 10 minutes.

e) /logo.svg:
   Cache-Control: public, max-age=2592000, immutable
   Reason: Static asset, never changes.
   30 days (but without content hash, avoid 1 year — risk if it ever changes).
   Better: content-hash the filename → use max-age=31536000, immutable.

f) /auth/token:
   Cache-Control: no-store
   Reason: Auth tokens are highly sensitive.
   Must NEVER be stored anywhere (memory or disk).

g) /images/hero.jpg:
   Cache-Control: public, max-age=86400
   Reason: Changes monthly — 24-hour cache is a reasonable tradeoff.
   CDN purge on campaign change is the invalidation mechanism.
   Alternative: content-hash in URL for immutable caching.
```

</details>

---

### Exercise 2 — Diagnose the caching problem

A user complains that after you deployed a CSS fix, they still see the old styles 2 hours later. Your current headers are:

```http
# Current headers on /styles.css
Cache-Control: public, max-age=3600
Last-Modified: [deployment timestamp]
```

1. Why is the user still seeing old styles?
2. What are two ways to fix this?
3. What's the best long-term solution?

<details>
<summary>Answer</summary>

```
1. Why old styles persist:
   max-age=3600 means the browser considers /styles.css fresh
   for 3600 seconds (1 hour) after it was cached.

   If the user last visited 1 hour ago, their cache has up to
   1 more hour of freshness — they won't revalidate until then.

   The browser has no mechanism to know you deployed new styles.
   Only time passing (max-age expiry) or a user hard-refresh
   (Ctrl+Shift+R) will trigger a re-download.

2. Two ways to fix immediately:

   a) Reduce max-age to a shorter duration + stale-while-revalidate:
      Cache-Control: public, max-age=60, stale-while-revalidate=3600
      After 60 seconds, revalidation happens. User gets fresh within 1 minute.

   b) Purge CDN cache for /styles.css:
      If you have a CDN, purge the cached version.
      New requests reach the origin with fresh Last-Modified headers.
      Browser's cached version expires and revalidates with new Last-Modified.
      → 304 if user cached the new version, 200 with new content if old.

3. Best long-term solution:
   Use content hashing:
   /styles.abc123.css — hash of content in filename
   Cache-Control: public, max-age=31536000, immutable

   HTML references /styles.abc123.css (the current hash).
   After deploy: new hash → new URL → browsers download fresh.
   No cache invalidation needed — URL IS the cache key.
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/08-critical-rendering-path.md`](./08-critical-rendering-path.md) — CRP optimization with caching
- [`caching/01-http-caching.md`](../caching/01-http-caching.md) — Advanced HTTP caching patterns
- [`caching/02-service-worker-cache.md`](../caching/02-service-worker-cache.md) — Service Worker caching in depth
- [`caching/03-memory-caching.md`](../caching/03-memory-caching.md) — In-memory caching patterns
- [`javascript-core/13-service-workers.md`](../javascript-core/13-service-workers.md) — Service Worker full guide

---

<div align="center">

**Next:** [`browser-internals/10-ssr-csr-isr-streaming.md`](./10-ssr-csr-isr-streaming.md) →

</div>
