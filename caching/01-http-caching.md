# 01 — HTTP Caching

> **"A cache miss means you paid the full price. A cache hit means you paid for something once and used it many times. A cache mistake means you paid the full price AND served wrong data. Getting caching right is the difference between a site that feels instant and one that feels broken."**

HTTP caching is the most impactful performance optimization available — a cache hit costs zero network time, zero server processing, and near-zero browser time. But HTTP caching also has the highest correctness risk of any optimization: a misconfigured cache can serve stale, incorrect, or security-sensitive data. This document covers advanced HTTP caching mechanics, edge caching, stale-while-revalidate, conditional requests, cache partitioning, and the design decisions that make caching safe as well as fast.

---

## 📚 Table of Contents

1. [The HTTP Cache Architecture](#1-the-http-cache-architecture)
2. [Cache-Control Deep Dive](#2-cache-control-deep-dive)
3. [Conditional Requests and Validation](#3-conditional-requests-and-validation)
4. [stale-while-revalidate in Practice](#4-stale-while-revalidate-in-practice)
5. [Cache Partitioning and Privacy](#5-cache-partitioning-and-privacy)
6. [CDN Caching vs Browser Caching](#6-cdn-caching-vs-browser-caching)
7. [Vary Header and Content Negotiation](#7-vary-header-and-content-negotiation)
8. [Caching API Responses](#8-caching-api-responses)
9. [Cache Invalidation Strategies](#9-cache-invalidation-strategies)
10. [Security Considerations](#10-security-considerations)
11. [Measuring Cache Performance](#11-measuring-cache-performance)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The HTTP Cache Architecture

### Cache Layers

```
USER REQUEST:

Browser → [SW Cache] → [Memory Cache] → [Disk Cache] → [CDN Cache] → [Origin Server]
         ↑ fastest                                                      ↑ slowest

1. Service Worker Cache (Cache API):
   Programmable, JavaScript-controlled
   Persists across sessions
   Can cache anything (non-standard responses, etc.)

2. Memory Cache (RAM):
   Sub-millisecond access
   Current browser session only
   Automatically populated for recent requests

3. Disk Cache (HTTP Cache):
   Millisecond access (disk read)
   Persists across sessions
   Respects HTTP cache headers (Cache-Control, ETag, etc.)

4. CDN / Proxy Cache:
   Separate from browser
   Serves many users from same cached copy
   Controlled by Cache-Control: s-maxage and CDN-specific headers

5. Origin Server:
   The authoritative source
   Should be hit as rarely as possible
```

### Request Resolution Flow

```javascript
// Conceptual flow for every request:
async function resolve(url) {
  // 1. Service Worker intercepts (if registered)
  const swResponse = await serviceWorkerCache.match(url);
  if (swResponse && !isExpired(swResponse)) return swResponse;

  // 2. Memory cache check
  const memResponse = memoryCache.get(url);
  if (memResponse && !isExpired(memResponse)) return memResponse;

  // 3. Disk cache check
  const diskEntry = diskCache.get(url);
  if (diskEntry) {
    if (isFresh(diskEntry)) return diskEntry.response;
    if (diskEntry.hasValidator) return conditionalRequest(url, diskEntry);
  }

  // 4. Network request (may hit CDN first)
  const response = await fetch(url);
  storeInCache(response);
  return response;
}
```

---

## 2. Cache-Control Deep Dive

### The Complete Header Reference

```http
# FRESHNESS DIRECTIVES
Cache-Control: max-age=3600
# Fresh for 3600 seconds from the response Date header
# Equivalent to (but supersedes) Expires header

Cache-Control: s-maxage=86400
# Overrides max-age for SHARED caches (CDN, Varnish, Nginx proxy)
# Browser ignores s-maxage, uses max-age
# CDN uses s-maxage, ignores max-age
# Combined: Cache-Control: max-age=3600, s-maxage=86400
# → Browser: fresh 1 hour, CDN: fresh 24 hours

Cache-Control: stale-while-revalidate=600
# After max-age expires: serve stale for up to 600 additional seconds
# while revalidating in background
# User gets instant response + freshness eventually
# See Section 4 for details

Cache-Control: stale-if-error=86400
# If revalidation fails (5xx, timeout): serve stale for 24 hours
# Graceful degradation: stale content > error page

# CACHEABILITY DIRECTIVES
Cache-Control: public
# Can be cached by browser AND shared caches (CDN, proxies)
# Default for most GET responses with max-age

Cache-Control: private
# Can ONLY be cached by browser (not CDN/proxy)
# Use for: user-specific responses
# Note: private does NOT mean encrypted — just not shared

Cache-Control: no-cache
# Store in cache but ALWAYS revalidate before serving
# Not "don't cache" — it's "cache but check every time"
# Sends conditional request on each use (304 if unchanged)

Cache-Control: no-store
# Do NOT cache at all — not in memory, not on disk
# Use for: sensitive data, auth tokens, PII
# Most secure, least performant

# TRANSFORM DIRECTIVES
Cache-Control: no-transform
# Intermediaries (CDN, proxy) must not modify the content
# Use for: images where CDN compression would be unwanted

# VALIDATION DIRECTIVES
Cache-Control: must-revalidate
# Once stale: MUST revalidate before serving
# Cannot use stale even if revalidation fails
# Stricter than default behavior

Cache-Control: proxy-revalidate
# Like must-revalidate but only for shared caches (CDN/proxy)

Cache-Control: immutable
# During max-age period: NEVER revalidate (even on F5/reload)
# For content-hashed URLs: hash in URL guarantees freshness
# Saves conditional request on user reload
# Only safe with content-hashed URLs (changing URL = new resource)
```

### Decision Matrix: Choosing the Right Cache-Control

| Resource Type           | Cache-Control                                    | Why                                           |
| ----------------------- | ------------------------------------------------ | --------------------------------------------- |
| HTML pages              | `no-cache`                                       | Must revalidate — references versioned assets |
| JS/CSS (content-hashed) | `public, max-age=31536000, immutable`            | URL changes on update                         |
| Images (versioned)      | `public, max-age=31536000, immutable`            | Same — content hashed                         |
| Images (unversioned)    | `public, max-age=86400`                          | Balance freshness/performance                 |
| Fonts                   | `public, max-age=31536000, immutable`            | Rarely change                                 |
| User-specific API data  | `private, no-cache`                              | Personal + always fresh                       |
| Public API data         | `public, max-age=60, stale-while-revalidate=600` | Shared, near-fresh                            |
| Auth tokens/PII         | `no-store`                                       | Never cache                                   |
| Service worker script   | `no-cache` (via updateViaCache: 'none')          | Always check for updates                      |

---

## 3. Conditional Requests and Validation

When a cached response expires, the browser can ask the server "has this changed?" rather than downloading it again.

### ETag-Based Validation

```
FIRST REQUEST:
  GET /api/users/list HTTP/1.1

  HTTP/1.1 200 OK
  Cache-Control: no-cache
  ETag: "users-v42-20240115"
  Content-Type: application/json
  [body: full users list]

  Browser stores: response + ETag value

SECOND REQUEST (cache expired or no-cache):
  GET /api/users/list HTTP/1.1
  If-None-Match: "users-v42-20240115"

  CASE 1 - Not modified:
  HTTP/1.1 304 Not Modified
  ETag: "users-v42-20240115"
  [no body — browser uses cached copy]
  Cost: ~200 bytes (headers only)

  CASE 2 - Modified:
  HTTP/1.1 200 OK
  ETag: "users-v43-20240116"
  [body: updated users list]
  Cost: full response
```

### Last-Modified Validation

```http
# Server includes last modified time:
Last-Modified: Mon, 15 Jan 2024 12:00:00 GMT

# Client sends on subsequent requests:
If-Modified-Since: Mon, 15 Jan 2024 12:00:00 GMT

# Server responds:
HTTP/1.1 304 Not Modified  # if unchanged
HTTP/1.1 200 OK            # if changed (with new Last-Modified)
```

### ETag Strength

```http
# Strong ETag: byte-for-byte identical content
ETag: "abc123def456"

# Weak ETag: semantically equivalent (acceptable for some uses)
ETag: W/"abc123def456"

# Strong ETags: exact content hash
# Generated by: hash of file content, version number, content hash
# Weak ETags: semantic equivalence
# Generated by: last-modified time, content categories

# For most use cases: use strong ETags
# If-None-Match only matches strong ETags by default
```

### Computing ETags in Node.js

```javascript
// Generate ETag from content hash (strong, stable)
import crypto from "crypto";

function generateETag(content) {
  return `"${crypto.createHash("md5").update(content).digest("hex")}"`;
}

// Express middleware:
app.use("/api/products", (req, res, next) => {
  const data = getProductsData();
  const etag = generateETag(JSON.stringify(data));

  // Check If-None-Match
  if (req.headers["if-none-match"] === etag) {
    return res.status(304).set("ETag", etag).end();
  }

  res
    .set({
      ETag: etag,
      "Cache-Control": "no-cache",
      "Content-Type": "application/json",
    })
    .json(data);
});
```

---

## 4. stale-while-revalidate in Practice

`stale-while-revalidate` is a powerful pattern: serve the cached response immediately while fetching a fresh copy in the background.

### How It Works

```
TIME AXIS:

t=0:     Resource first cached
         max-age=60, stale-while-revalidate=300

t=0→60:  FRESH zone
         Serve from cache immediately
         No network request

t=60→360: STALE-WHILE-REVALIDATE zone (60+300=360)
         Serve stale immediately (fast response!)
         Start background fetch for fresh version
         Next request: gets the updated version

t=360+:  STALE (beyond stale-while-revalidate)
         Must revalidate synchronously before serving
         User waits for network (slow response)
```

```http
# Ideal for: data that changes but where brief staleness is acceptable
# Examples: product catalog, news feed, dashboard metrics

Cache-Control: public, max-age=60, stale-while-revalidate=300
# Users always get fast responses (< 300s old)
# Server load: at most 1 request per 60s per cache entry
# Freshness: data is at most 360s old at the worst case
```

### JavaScript API: stale-while-revalidate Pattern

```javascript
// Service Worker implementation:
self.addEventListener("fetch", (event) => {
  if (event.request.method !== "GET") return;

  event.respondWith(
    caches.open("api-cache").then(async (cache) => {
      const cached = await cache.match(event.request);

      if (cached) {
        const age = getAge(cached); // seconds since cached
        const cacheControl = parseCacheControl(
          cached.headers.get("Cache-Control"),
        );
        const maxAge = cacheControl["max-age"] ?? 0;
        const swr = cacheControl["stale-while-revalidate"] ?? 0;

        if (age < maxAge) {
          // FRESH: return immediately, no revalidation
          return cached;
        }

        if (age < maxAge + swr) {
          // STALE-WHILE-REVALIDATE: return stale, revalidate in background
          event.waitUntil(
            fetch(event.request).then((response) => {
              cache.put(event.request, response.clone());
            }),
          );
          return cached; // immediate stale response
        }
      }

      // No cache or expired: fetch synchronously
      const response = await fetch(event.request);
      cache.put(event.request, response.clone());
      return response;
    }),
  );
});

function getAge(response) {
  const date = response.headers.get("Date");
  if (!date) return Infinity;
  return (Date.now() - new Date(date).getTime()) / 1000;
}
```

### Application-Level stale-while-revalidate (TanStack Query)

```typescript
// TanStack Query implements stale-while-revalidate natively
useQuery({
  queryKey: ["products"],
  queryFn: () => productsApi.list(),
  staleTime: 60_000, // serve cached for up to 60s without refetching
  gcTime: 300_000, // keep in cache for 5 minutes even after staleTime
  // After 60s: serves cached data immediately + refetches in background
  // After 300s: garbage collected
});
```

---

## 5. Cache Partitioning and Privacy

Modern browsers partition caches by site to prevent cross-site tracking.

### The Problem Cache Partitioning Solves

```
BEFORE CACHE PARTITIONING (pre-2020):
  Site A loads:  https://cdn.example.com/react.js
  → Cached globally in browser cache

  Site B loads:  https://cdn.example.com/react.js
  → Cache HIT! Served from cache (fast)
  → SIDE EFFECT: Site B can detect Site A was visited
    (by measuring if react.js loads instantly or takes time)
  → Browser cache timing attacks became possible

AFTER CACHE PARTITIONING (2020+):
  Cache key = (site A, https://cdn.example.com/react.js)
  Cache key = (site B, https://cdn.example.com/react.js)
  → Different cache keys → different entries → no cross-site sharing
```

### Impact on CDN Caching Benefits

```
BEFORE PARTITIONING:
  100 sites use React from CDN → cached once → all benefit
  Cache hit rate: very high for popular resources

AFTER PARTITIONING:
  Each (site, resource) pair is a separate cache entry
  Same React file from CDN is cached per-site, not globally

  Performance impact:
  - First visit to any site with React CDN: cache miss
  - Return visits: cache hit (same-site partitioning preserved)
  - Cross-site sharing: eliminated

  Practical implication:
  - Self-host critical resources (no longer benefit from "CDN warming")
  - Inline critical CSS/fonts to avoid extra requests
  - Use connection preconnect to reduce CDN connection cost
```

---

## 6. CDN Caching vs Browser Caching

Understanding the difference between these two layers is crucial for correct cache strategy.

### Key Differences

```
BROWSER CACHE:
  Location:   User's device
  Scope:      One user, one browser
  Control:    Cache-Control (browser respects private)
  Invalidate: User clears cache, or max-age expires
  Use for:    Per-user performance

CDN CACHE:
  Location:   CDN edge nodes (globally distributed)
  Scope:      ALL users hitting that CDN edge
  Control:    Cache-Control (s-maxage for CDN, max-age for browser)
  Invalidate: CDN purge API, or s-maxage expires
  Use for:    Origin server load reduction, global performance
```

### Split Cache Headers

```http
# Different TTL for browser vs CDN:
Cache-Control: max-age=300, s-maxage=86400

# Browser caches for 5 minutes
# CDN caches for 24 hours

# Why? CDN can serve millions of requests from its cache (high efficiency)
# Browser sees stale after 5 minutes → revalidates against CDN (not origin)
# CDN can serve the validation response if unchanged (304) → minimal origin load

# For deployments: purge CDN on deploy
# Browser cache expires naturally (< 5 min stale at worst)
```

### CDN-Specific Headers

```http
# Cloudflare: control CDN behavior beyond Cache-Control
CDN-Cache-Control: max-age=86400
# Cloudflare respects this over Cache-Control: max-age for CDN layer

# Surrogate-Control (Varnish, Fastly, Akamai):
Surrogate-Control: max-age=86400
# Fastly: strip this header before forwarding to browser
# Browser only sees Cache-Control

# Cache-Tag (CDN resource tagging for bulk invalidation):
Cache-Tag: product-42, category-electronics
# Purge all resources with tag "product-42" in one API call
```

### Programmatic CDN Cache Purge

```javascript
// Purge specific URLs on deploy
async function purgeCDNCache(urls) {
  // Cloudflare example:
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/purge_cache`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${CF_API_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ files: urls }),
    },
  );
  return response.json();
}

// In CI/CD after deployment:
await purgeCDNCache([
  "https://example.com/",
  "https://example.com/products",
  "https://example.com/sitemap.xml",
]);
// Purge HTML pages (which reference versioned assets)
// Assets (JS/CSS/images): content-hashed → no purge needed
```

---

## 7. Vary Header and Content Negotiation

The `Vary` header tells caches that the response depends on specific request headers.

### How Vary Works

```
Without Vary:
  GET /api/products
  Accept-Encoding: gzip
  → Cache stores response

  GET /api/products
  Accept-Encoding: br (Brotli)
  → Cache HIT — serves gzip response to client expecting Brotli!
  → Incorrect behavior

With Vary: Accept-Encoding:
  GET /api/products (Accept-Encoding: gzip)
  → Cache key: "/api/products + gzip"
  → Stores gzip response

  GET /api/products (Accept-Encoding: br)
  → Cache key: "/api/products + br"
  → Cache MISS — fetches Brotli response separately
  → Correct behavior: each encoding cached separately
```

### Vary Header Combinations

```http
# Common: vary by encoding (universal)
Vary: Accept-Encoding

# Language negotiation
Vary: Accept-Language
# Cache stores separate response for each language
# Warning: can multiply cache entries significantly

# Content type negotiation
Vary: Accept
# API serving JSON vs XML based on Accept header

# Multiple headers
Vary: Accept-Encoding, Accept-Language
# Cache key = (url + encoding + language)
# Warning: combinatorial explosion of cache entries

# Kill CDN caching (avoid unless necessary)
Vary: Cookie
# Cookie is different per user → CDN can't share cached responses
# Effectively disables CDN caching for this resource
# Use Cache-Control: private instead for user-specific content

Vary: Authorization
# Authorization token different per user → CDN can't cache
# Again: use Cache-Control: private for user-specific content
```

### Vary Optimization

```javascript
// Server: normalize headers to reduce Vary cache fragmentation

// ❌ Vary on raw Accept-Language: "en-US,en;q=0.9,fr;q=0.8"
// Different users send different strings → many cache entries

// ✅ Normalize to canonical locale before caching
function getLocale(req) {
  const lang = req.headers["accept-language"];
  // "en-US,en;q=0.9,fr;q=0.8" → "en"
  return lang.split(",")[0].split("-")[0];
}

// Use in cache key instead of raw header:
res.set("Vary", "Accept-Encoding"); // not Accept-Language
// Add locale to the URL instead: /api/products?locale=en
// URL-based locale = clean cache keys
```

---

## 8. Caching API Responses

API responses have unique caching challenges compared to static assets.

### When to Cache API Responses

```
CACHE-FRIENDLY (public data, tolerable staleness):
  ✓ Public product catalogs
  ✓ News articles
  ✓ Configuration data
  ✓ Reference data (countries, currencies, categories)

NOT CACHE-FRIENDLY:
  ✗ Auth tokens (no-store)
  ✗ User-specific profiles (private)
  ✗ Real-time inventory/pricing (no-cache)
  ✗ Financial transactions (no-store)
```

### API Cache Headers by Endpoint Type

```javascript
// Express.js: cache headers per endpoint type

// Public reference data: cache aggressively
app.get("/api/countries", cacheMiddleware({ maxAge: 86400, sMaxAge: 604800 }));
// Browser: 1 day, CDN: 7 days

// Public product catalog: moderate cache
app.get(
  "/api/products",
  cacheMiddleware({
    maxAge: 60,
    sMaxAge: 3600,
    staleWhileRevalidate: 600,
  }),
);

// User-specific data: private, short cache
app.get(
  "/api/user/profile",
  authenticate,
  cacheMiddleware({
    maxAge: 30,
    private: true,
    noCache: true, // revalidate always
  }),
);

// Auth: never cache
app.post("/api/auth/token", (req, res) => {
  res.set("Cache-Control", "no-store");
  // ...
});

function cacheMiddleware({
  maxAge,
  sMaxAge,
  staleWhileRevalidate,
  private: priv,
  noCache,
  noStore,
}) {
  return (req, res, next) => {
    if (req.method !== "GET") {
      res.set("Cache-Control", "no-store");
      return next();
    }

    const directives = [];
    if (noStore) {
      directives.push("no-store");
    } else if (noCache) {
      directives.push("no-cache");
    } else {
      directives.push(priv ? "private" : "public");
      if (maxAge !== undefined) directives.push(`max-age=${maxAge}`);
      if (sMaxAge !== undefined && !priv)
        directives.push(`s-maxage=${sMaxAge}`);
      if (staleWhileRevalidate !== undefined)
        directives.push(`stale-while-revalidate=${staleWhileRevalidate}`);
    }

    res.set("Cache-Control", directives.join(", "));
    next();
  };
}
```

### ETag for API Responses

```javascript
// Compute ETag from response content
import crypto from "crypto";

function contentETag(data) {
  const json = typeof data === "string" ? data : JSON.stringify(data);
  return `"${crypto.createHash("sha1").update(json).digest("hex").slice(0, 16)}"`;
}

app.get("/api/products", (req, res) => {
  const products = getProducts(req.query);
  const etag = contentETag(products);

  // Check conditional request
  if (req.headers["if-none-match"] === etag) {
    return res
      .status(304)
      .set("ETag", etag)
      .set("Cache-Control", "public, max-age=60, stale-while-revalidate=600")
      .end();
  }

  res
    .set({
      ETag: etag,
      "Cache-Control": "public, max-age=60, stale-while-revalidate=600",
      "Content-Type": "application/json",
    })
    .json(products);
});
```

---

## 9. Cache Invalidation Strategies

The hardest problem in caching: knowing when to invalidate.

### Strategy 1 — Content-Based Cache Busting (Immutable)

```
URL includes content hash → URL changes when content changes

/assets/app.abc123.js
  Cache-Control: public, max-age=31536000, immutable

After deploy:
/assets/app.xyz789.js (new hash → new URL → browsers fetch fresh)

Old URL cached forever (but never referenced after deploy)
→ No invalidation needed
→ Old entries slowly evicted by LRU
```

### Strategy 2 — Version in URL

```
/api/v2/products (URL includes API version)
Cache-Control: public, max-age=3600

Breaking API change: create /api/v3/products
Old clients still use v2 cache (unaffected)
New clients use v3 (separate cache entry)
```

### Strategy 3 — CDN Tag-Based Invalidation

```javascript
// Tag resources with logical identifiers
// Purge by tag on data change

// On response:
res.set("Cache-Tag", `product-${product.id} category-${product.category}`);
res.set("Cache-Control", "public, max-age=3600");

// When product 42 is updated:
await cloudflare.purgeByTag("product-42");
// All responses tagged with product-42 are invalidated globally

// When a category changes:
await cloudflare.purgeByTag("category-electronics");
// All product listings for electronics are invalidated
```

### Strategy 4 — Short max-age + stale-while-revalidate

```http
# Practical for frequently changing content where CDN purge API isn't set up
Cache-Control: public, max-age=60, stale-while-revalidate=300

# Worst case staleness: 360 seconds (6 minutes)
# Always-fast response: served from cache immediately
# No explicit invalidation needed — expires naturally
```

### Strategy 5 — On-Demand Revalidation (ISR pattern)

```javascript
// Next.js-style on-demand revalidation
// Keep long cache, but allow instant purge via webhook

// Response header: long CDN cache
res.set(
  "Cache-Control",
  "public, s-maxage=86400, stale-while-revalidate=86400",
);

// Revalidation endpoint (called by CMS webhook on content change):
app.post("/api/revalidate", async (req, res) => {
  if (req.headers["x-revalidate-secret"] !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ error: "Unauthorized" });
  }

  const { path } = req.body;

  // Purge CDN cache for this path
  await cdnClient.purge([`https://example.com${path}`]);

  res.json({ revalidated: true, path });
});
```

---

## 10. Security Considerations

### Never Cache Sensitive Data

```http
# Auth tokens, session data, personal information
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache (legacy fallback)
Expires: 0 (legacy fallback)

# Also set:
Surrogate-Control: no-store  # Prevent CDN caching
```

### Cache Poisoning Prevention

```javascript
// Cache Poisoning: attacker tricks cache into storing malicious response
// that gets served to other users

// VULNERABILITY: caching based on unvalidated input
app.get('/search', (req, res) => {
  const query = req.query.q;
  res.set('Cache-Control', 'public, max-age=3600');
  // ❌ If X-Forwarded-Host can manipulate the response:
  // Attacker sends: X-Forwarded-Host: evil.com
  // Cache stores response with evil.com references
  // Other users get the poisoned response
});

// PREVENTION:
// 1. Don't cache responses that vary on unvalidated request headers
// 2. Use Vary: only for well-defined, validated headers
// 3. CDN: deny/normalize unrecognized request headers
// 4. Don't reflect user input into cached responses

// Nginx: strip unexpected headers before caching
proxy_ignore_headers X-Accel-Expires;
# Only cache responses where the origin explicitly set cache headers
```

### Sensitive Headers in Cache Keys

```http
# Authorization: NEVER include in cache key (would cache per-token)
# If you must cache authenticated responses: use s-maxage=0 for CDN
# and Cache-Control: private for browser-only caching

# NEVER do:
Vary: Authorization
Cache-Control: public, max-age=3600
# This creates separate cache entries per auth token
# If a token is compromised: its cached data persists in CDN

# DO:
Cache-Control: private, max-age=30, no-cache
# Browser-only, always revalidated
```

---

## 11. Measuring Cache Performance

### Cache Hit Rate

```javascript
// Server-side: log cache outcomes
app.use((req, res, next) => {
  const start = Date.now();

  res.on("finish", () => {
    const cacheStatus =
      res.getHeader("X-Cache-Status") ||
      (res.statusCode === 304 ? "REVALIDATED" : "MISS");

    metrics.record("http_cache", {
      url: req.url,
      status: cacheStatus, // HIT, MISS, REVALIDATED, STALE
      duration_ms: Date.now() - start,
      status_code: res.statusCode,
    });
  });

  next();
});

// CDN: typically exposes cache status headers
// X-Cache: HIT | MISS (Cloudflare, Varnish)
// CF-Cache-Status: HIT | MISS | EXPIRED | REVALIDATED | UPDATING (Cloudflare)
// X-Served-By: cache-... (Fastly)
```

### Client-Side Cache Measurement

```javascript
// PerformanceResourceTiming: detect cache hits
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType !== "resource") continue;

    const isCached = entry.transferSize === 0 && entry.decodedBodySize > 0;
    const isConditional = entry.transferSize < 300 && entry.decodedBodySize > 0;
    // transferSize < 300 bytes: likely a 304 Not Modified response (headers only)

    metrics.track("resource_cache", {
      url: entry.name,
      cached: isCached,
      validated: isConditional,
      size: entry.decodedBodySize,
      duration: entry.duration,
    });
  }
});
observer.observe({ entryTypes: ["resource"] });
```

---

## 12. Good Practices

### ✅ Content hash all static assets

```javascript
// Webpack/Vite output configuration:
output: {
  filename: "[name].[contenthash:8].js";
}
// Enables: Cache-Control: public, max-age=31536000, immutable
```

### ✅ Separate CDN and browser TTLs

```http
Cache-Control: max-age=300, s-maxage=86400
# Browser: 5 minutes (user sees fresh data quickly after changes)
# CDN: 24 hours (minimize origin load)
# CDN purge on deploy handles invalidation
```

### ✅ Use ETag + no-cache for HTML

```http
# HTML: must be fresh (references versioned assets)
# ETag: cheap validation (304 if unchanged)
Cache-Control: no-cache
ETag: "page-hash-abc"
# → every load: conditional request, but only downloads if HTML changed
```

### ✅ Monitor CDN cache hit rate

```
Target: > 90% cache hit rate for static assets
Target: > 80% cache hit rate for API responses (public endpoints)
If below: investigate cache busting issues, missing cache headers,
          Vary header fragmentation, or TTL too short
```

---

## 13. Bad Practices

### ❌ `no-cache` for static assets with content hashing

```http
# ❌ Wasted conditional request — URL already guarantees freshness
GET /app.abc123.js
Cache-Control: no-cache
# → conditional request every reload → server CPU wasted

# ✅ Correct for content-hashed assets:
Cache-Control: public, max-age=31536000, immutable
```

### ❌ Long cache for mutable URLs

```http
# ❌ Main CSS file with long cache but no URL versioning
GET /styles.css
Cache-Control: max-age=86400
# After deploy: users see old styles for up to 24 hours
# No way to invalidate browser cache
```

### ❌ Caching user-specific data at the CDN level

```http
# ❌ User profile cached at CDN — served to wrong user
GET /api/user/profile
Cache-Control: public, max-age=300
# CDN serves user A's profile to user B if they're on same edge node
# Serious privacy/security violation

# ✅ User-specific must be private
Cache-Control: private, max-age=30
```

---

## 14. Common Mistakes

### Mistake 1 — Confusing no-cache and no-store

```
no-cache: "Cache this, but validate before serving"
  → Stored on disk
  → Conditional GET on each use
  → 304 if unchanged (fast)
  → 200 with body if changed

no-store: "Do NOT store this anywhere"
  → Not stored
  → Full download every time
  → Most secure, most expensive

Use no-cache for: HTML, frequently-updated data
Use no-store for: sensitive data (auth, PII, financial)
```

### Mistake 2 — Forgetting the Age header

```http
# Age header: seconds since the response was generated/last validated
# Key for understanding cache freshness

Cache-Control: max-age=3600
Age: 2400
# Resource was cached 2400 seconds ago
# Remaining freshness: 3600 - 2400 = 1200 seconds (20 minutes)

# If Age > max-age: resource is stale (even if just received from CDN)
# The CDN's cache has been holding this for 2400s

# Age header is set by CDN/proxy — not by origin server
# Helps diagnose "why is this stale?"
```

### Mistake 3 — Not validating `Vary` behavior

```javascript
// Test that Vary headers actually work correctly
async function testVaryBehavior(url) {
  const responses = await Promise.all([
    fetch(url, { headers: { "Accept-Encoding": "gzip" } }),
    fetch(url, { headers: { "Accept-Encoding": "br" } }),
    fetch(url, { headers: { "Accept-Language": "en" } }),
    fetch(url, { headers: { "Accept-Language": "fr" } }),
  ]);

  const varyValues = responses.map((r) => r.headers.get("Vary"));
  const contentEncodings = responses.map((r) =>
    r.headers.get("Content-Encoding"),
  );

  console.table({ varyValues, contentEncodings });
  // Verify: gzip request got gzip response, br request got br response
}
```

---

## 15. Interview-Level Explanation

> **"How does HTTP caching work? How do you design a cache strategy for a web application?"**

**Strong answer:**

> "HTTP caching has two distinct layers: the browser's disk cache and CDN/shared proxy caches. The Cache-Control header governs both, with `max-age` controlling browser freshness and `s-maxage` controlling CDN freshness independently. `public` allows CDN caching; `private` restricts to browser only.
>
> The caching strategy for a typical web application follows a clear pattern by resource type. Static assets with content-hashed filenames — like `app.abc123.js` — get `Cache-Control: public, max-age=31536000, immutable`. The URL changes when content changes, so the cache is effectively never stale. `immutable` tells the browser not to send conditional requests even on reload. HTML pages get `no-cache` with an ETag — they must always be validated since they reference the versioned asset URLs, but if unchanged a 304 Not Modified costs only headers. User-specific API data gets `private` to prevent CDN caching. Sensitive data gets `no-store`.
>
> The difference between `no-cache` and `no-store` trips people up. `no-cache` means 'store it, but always revalidate before serving.' The browser sends `If-None-Match` on every request; if unchanged, the server returns 304 with no body. `no-store` means 'don't store this at all' — full download every time. Use `no-store` only for genuinely sensitive data.
>
> `stale-while-revalidate` is a critical pattern for APIs. With `max-age=60, stale-while-revalidate=300`, requests always get an immediate cache response, but after 60 seconds the browser revalidates in the background. Users never wait for the network; data is at most 360 seconds old at worst.
>
> Cache invalidation for mutable URLs requires proactive work — either CDN purge APIs (purge specific URLs on deploy), tag-based purging (tag all product-42 resources and purge by tag when product 42 changes), or short TTLs plus stale-while-revalidate. Content-hashed URLs are the cleanest because they require no invalidation logic at all."

---

## 16. Exercises

### Exercise 1 — Cache headers audit

Given these HTTP responses, identify what's wrong with each cache strategy:

```
a) GET /index.html
   Cache-Control: max-age=86400

b) GET /app.v1.js (content-hashed)
   Cache-Control: no-cache

c) GET /api/user/dashboard
   Cache-Control: public, max-age=3600

d) GET /api/auth/token (POST response returning token)
   Cache-Control: max-age=300

e) GET /api/products (public catalog)
   Cache-Control: private, no-cache
```

<details>
<summary>Answers</summary>

```
a) GET /index.html — Cache-Control: max-age=86400
   Problem: HTML cached for 24 hours with NO validation
   After deploy: users see old HTML (old asset URLs) for up to 24 hours
   No ETag/no-cache = no way to serve fresh immediately
   Fix: Cache-Control: no-cache (+ ETag for cheap validation)

b) GET /app.v1.js — Cache-Control: no-cache
   Problem: Content-hashed asset with revalidation
   URL already guarantees freshness (hash changes with content)
   no-cache forces a conditional request on every reload — wasted round trip
   Fix: Cache-Control: public, max-age=31536000, immutable

c) GET /api/user/dashboard — Cache-Control: public, max-age=3600
   Problem: User-specific data cached publicly
   CDN will serve User A's dashboard to User B (severe privacy violation)
   Fix: Cache-Control: private, max-age=30, no-cache
   (or no-store if very sensitive)

d) POST /api/auth/token — Cache-Control: max-age=300
   Problem: Auth token in a cacheable response
   POST responses should not be cached
   Auth tokens should NEVER be cached (no-store)
   Fix: Cache-Control: no-store

e) GET /api/products — Cache-Control: private, no-cache
   Problem: Public catalog data unnecessarily private
   Cannot benefit from CDN caching — every user request hits origin
   no-cache forces revalidation on every request
   Fix: Cache-Control: public, max-age=60, stale-while-revalidate=600
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/09-browser-caching.md`](../browser-internals/09-browser-caching.md) — Browser cache fundamentals
- [`caching/02-service-worker-cache.md`](./02-service-worker-cache.md) — Service Worker caching strategies
- [`caching/05-cdn-strategies.md`](./05-cdn-strategies.md) — CDN caching in depth
- [`browser-internals/08-critical-rendering-path.md`](../browser-internals/08-critical-rendering-path.md) — Caching impact on CRP
- [`security/03-headers.md`](../security/03-headers.md) — Security headers including cache control for sensitive data

---

<div align="center">

**Next:** [`caching/02-service-worker-cache.md`](./02-service-worker-cache.md) →

</div>
