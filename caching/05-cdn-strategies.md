# 05 — CDN Strategies

> **"A CDN is a geographic extension of your server. Instead of every user in Tokyo hitting your server in Virginia, they hit an edge node 5 miles away. The latency drops from 180ms to 8ms. That's the CDN's primary job — not bandwidth savings, not protection, not features. Proximity. Everything else is secondary."**

A Content Delivery Network distributes cached copies of your content across geographically distributed edge nodes. Users are served from the node nearest to them, reducing latency, offloading origin traffic, and improving reliability. This document covers CDN architecture, edge caching strategies, cache-key design, edge functions, image optimization via CDN, CDN selection, and the operational patterns that make CDNs effective in production.

---

## 📚 Table of Contents

1. [CDN Architecture](#1-cdn-architecture)
2. [What to Put on a CDN](#2-what-to-put-on-a-cdn)
3. [Cache Key Design](#3-cache-key-design)
4. [CDN Cache-Control Headers](#4-cdn-cache-control-headers)
5. [Purge and Invalidation](#6-purge-and-invalidation)
6. [Edge Functions — Compute at the Edge](#6-edge-functions--compute-at-the-edge)
7. [Image Optimization via CDN](#7-image-optimization-via-cdn)
8. [CDN and Security](#8-cdn-and-security)
9. [CDN Performance Tuning](#9-cdn-performance-tuning)
10. [Multi-CDN Architecture](#10-multi-cdn-architecture)
11. [Measuring CDN Performance](#11-measuring-cdn-performance)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. CDN Architecture

### How a CDN Request Works

```
WITHOUT CDN:
  User in Tokyo → 180ms → Origin Server (Virginia)
  TTFB: 180ms + server processing time

WITH CDN:
  User in Tokyo → 8ms → Edge Node (Tokyo) → Cache HIT: 8ms total
                                            → Cache MISS: 8ms + 180ms + store
  After warm-up: all Tokyo users get < 10ms TTFB

CDN TOPOLOGY:
  ┌─────────────────────────────────────────────────────────────────┐
  │   ORIGIN (your server)                                          │
  │   e.g., AWS us-east-1, GCP europe-west1                        │
  └─────────────────────────┬───────────────────────────────────────┘
                            │ (cache miss: CDN fetches from origin)
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
     │ Edge Node   │ │ Edge Node   │ │ Edge Node   │
     │ New York    │ │ London      │ │ Singapore   │
     └─────────────┘ └─────────────┘ └─────────────┘
            ▲               ▲               ▲
     Users  │       Users   │      Users    │
     (East  │       (Europe)│      (Asia)   │
      USA)  │               │               │
```

### CDN Tiers

```
TIER 1 (PoPs — Points of Presence):
  Edge nodes closest to users
  100-300 locations globally (Cloudflare: 310+, Akamai: 4000+)
  Serves cached content

TIER 2 (Mid-tier / Shield):
  Intermediate cache layer between Tier 1 and origin
  Reduces origin traffic by sharing a single cache
  "Origin Shield" — all Tier 1 misses go through one Shield node
  Example: All North American edges share one shield in Ashburn, VA

ORIGIN:
  Your server(s)
  Only hit on cache misses that bypass all CDN tiers

BENEFIT OF ORIGIN SHIELD:
  Without shield: 300 edges each fetch the same resource from origin = 300 origin requests
  With shield: 300 edges miss → 1 shield node fetches → caches → serves all 300 edges = 1 origin request
```

---

## 2. What to Put on a CDN

### Cache Everything Possible

```
HIGH VALUE (cache with long TTL):
  ✓ Static assets (JS, CSS, fonts, images) with content hashing
  ✓ Versioned media files
  ✓ Downloadable files (PDFs, archives)
  ✓ Public API responses (product catalog, pricing, content)
  ✓ Static HTML pages (landing pages, documentation)
  ✓ Video/audio files

MEDIUM VALUE (cache with shorter TTL or SWR):
  ✓ HTML pages that change periodically
  ✓ RSS/Atom feeds
  ✓ Public aggregate data (rankings, leaderboards)
  ✓ Sitemap.xml

DO NOT CACHE AT CDN LEVEL:
  ✗ Authenticated user-specific responses (use Cache-Control: private)
  ✗ Responses containing cookies or session data
  ✗ Responses with Vary: Cookie (each user gets different response)
  ✗ Real-time data (stock prices, live scores, live positions)
  ✗ Payment processing endpoints
  ✗ Authentication tokens or sessions
  ✗ Streaming responses (SSE, WebSocket upgrades)
```

### CDN Cache-Ability Decision Tree

```javascript
function shouldCDNCache(request, response) {
  // Never cache non-GET
  if (!["GET", "HEAD"].includes(request.method)) return false;

  // Never cache if marked private
  const cacheControl = response.headers["cache-control"] ?? "";
  if (cacheControl.includes("private")) return false;
  if (cacheControl.includes("no-store")) return false;

  // Never cache authenticated responses
  if (request.headers["authorization"]) return false;
  if (request.headers["cookie"] && !cacheControl.includes("s-maxage"))
    return false;

  // Cache if explicitly allowed
  if (
    cacheControl.includes("s-maxage") ||
    cacheControl.includes("max-age") ||
    response.headers["expires"]
  )
    return true;

  // Default: don't cache (be conservative)
  return false;
}
```

---

## 3. Cache Key Design

The cache key determines when two requests get the same cached response. Poor key design causes bugs or poor cache hit rates.

### Default Cache Key

```
Default CDN cache key: scheme + host + path + query string
  https://example.com/products?page=1&sort=price → unique cache entry
  https://example.com/products?sort=price&page=1 → DIFFERENT cache entry (same content!)
```

### Normalizing Query Parameters

```nginx
# Cloudflare: Cache Rules → Cache Key → query string rules
# Or: Cloudflare Workers to normalize keys

# Sort query params before caching
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);

  // Normalize query string for cache key
  const sortedParams = new URLSearchParams(
    [...url.searchParams.entries()].sort(([a], [b]) => a.localeCompare(b))
  );
  url.search = sortedParams.toString();

  // Remove ignored params from cache key
  const IGNORED_PARAMS = ['utm_source', 'utm_medium', 'utm_campaign', 'fbclid', 'gclid'];
  IGNORED_PARAMS.forEach(p => url.searchParams.delete(p));

  // Fetch with normalized URL
  return fetch(new Request(url.toString(), request));
}
```

### Custom Cache Keys

```nginx
# Nginx proxy_cache_key: customize what uniquely identifies a response
proxy_cache_key "$scheme$request_method$host$request_uri$http_accept_encoding";
# → adds encoding to key so gzip and brotli are cached separately

# Varnish: custom hash
sub vcl_hash {
  hash_data(req.url);
  hash_data(req.http.host);
  # Add Accept-Language normalized to top language only
  if (req.http.Accept-Language ~ "^en") {
    hash_data("en");
  } elsif (req.http.Accept-Language ~ "^fr") {
    hash_data("fr");
  } else {
    hash_data("en"); # default
  }
  return(lookup);
}
```

### Cache Key for API Responses

```javascript
// Cloudflare Worker: custom cache key for API requests
async function handleApiRequest(request) {
  const url = new URL(request.url);
  const cache = caches.default;

  // Build custom cache key
  const cacheKey = new Request(
    `${url.origin}/api-cache${url.pathname}?${buildCacheKeyParams(url.searchParams)}`,
    { method: "GET", headers: { "Cache-Control": "no-transform" } },
  );

  // Check CDN cache
  const cached = await cache.match(cacheKey);
  if (cached) return cached;

  // Fetch from origin
  const response = await fetch(request);

  // Store with cache key and appropriate TTL
  if (response.ok) {
    const responseToCache = new Response(response.body, response);
    responseToCache.headers.set("Cache-Control", "public, s-maxage=300");
    await cache.put(cacheKey, responseToCache.clone());
  }

  return response;
}

function buildCacheKeyParams(params) {
  // Only include parameters that affect response content
  const CACHE_KEY_PARAMS = ["page", "limit", "sort", "category", "format"];
  const filtered = new URLSearchParams();
  CACHE_KEY_PARAMS.forEach((p) => {
    if (params.has(p)) filtered.set(p, params.get(p));
  });
  return filtered.toString();
}
```

---

## 4. CDN Cache-Control Headers

CDN-specific cache control:

```http
# Standard: both browser and CDN
Cache-Control: public, max-age=300, s-maxage=86400
# Browser: fresh 5 minutes
# CDN: fresh 24 hours

# CDN-only (Cloudflare-specific):
CDN-Cache-Control: max-age=86400
# Cloudflare uses this, ignores Cache-Control: max-age for its own TTL
# Browser only sees Cache-Control

# Fastly:
Surrogate-Control: max-age=86400
# Fastly strips this before forwarding to browser
# Browser never sees it

# Akamai:
Edge-Control: max-age=86400
# Akamai-specific, stripped before browser

# Cloudflare Cache-Status diagnostic header (in responses):
CF-Cache-Status: HIT   # served from Cloudflare cache
CF-Cache-Status: MISS  # cache miss, fetched from origin
CF-Cache-Status: REVALIDATED  # was stale, revalidated with origin
CF-Cache-Status: UPDATING  # stale, background update in progress
CF-Cache-Status: EXPIRED   # expired, served stale while revalidating
```

---

## 5. Purge and Invalidation

### URL-Based Purge

```javascript
// Cloudflare: purge specific URLs on deploy
async function purgeURLs(urls) {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/purge_cache`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${CF_API_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ files: urls }),
    },
  );
  const result = await response.json();
  if (!result.success)
    throw new Error(`Purge failed: ${JSON.stringify(result.errors)}`);
  console.log(`Purged ${urls.length} URLs`);
  return result;
}

// Call during CI/CD after successful deployment:
await purgeURLs([
  "https://example.com/",
  "https://example.com/products",
  "https://example.com/blog",
  "https://example.com/sitemap.xml",
]);
// Assets (JS/CSS) have content-hashed URLs → don't need purging
```

### Tag-Based Purge (Surrogate Keys)

```javascript
// Tag resources with semantic identifiers
// Allows batch invalidation by business concept

// In your API server responses:
app.get("/products/:id", (req, res) => {
  const product = getProduct(req.params.id);

  // Tag this response with semantic identifiers
  res.set(
    "Cache-Tag",
    [
      `product-${product.id}`,
      `category-${product.categoryId}`,
      `brand-${product.brandId}`,
    ].join(","),
  );

  res.set("Cache-Control", "public, s-maxage=3600");
  res.json(product);
});

// When product 42 is updated: purge only product-42 tagged responses
async function onProductUpdated(productId) {
  // Purge all CDN responses tagged with this product
  await cloudflare.purgeTags([`product-${productId}`]);
  // Invalidates: product detail pages, product-in-list pages, etc.
  // Does NOT purge unrelated content
}

// When a category changes: purge all products in that category
async function onCategoryChanged(categoryId) {
  await cloudflare.purgeTags([`category-${categoryId}`]);
}
```

### Stale-While-Revalidate at CDN Level

```http
# CDN stale-while-revalidate: serve stale while CDN refetches from origin
Cache-Control: public, s-maxage=60, stale-while-revalidate=600

# CDN behavior:
# t=0:   Fresh from origin, stored in CDN
# t=60:  Expired (s-maxage=60)
# t=61:  Request arrives: CDN serves STALE immediately
#        CDN background-fetches from origin
# t=360: Request arrives during stale period: still serves stale
#        CDN continuously checking for freshness
# t=660: Beyond stale-while-revalidate: must wait for fresh response

# Result: users never see latency due to CDN cache expiry
# (CDN handles revalidation transparently)
```

---

## 6. Edge Functions — Compute at the Edge

Edge functions run JavaScript at CDN edge nodes — near the user, before the request hits the origin.

### Use Cases for Edge Functions

```
AUTHENTICATION: Verify JWT at edge — reject unauthorized requests without hitting origin
GEOLOCATION: Redirect users to regional endpoints based on IP location
A/B TESTING: Assign variants at edge — no round trip to origin
PERSONALIZATION: Add user context to cache key or modify response
RATE LIMITING: Block excessive requests at the edge
REDIRECTS: Handle URL redirects at edge — no origin request needed
HEADER MANIPULATION: Add/remove headers, normalize requests
ROBOTS.TXT GENERATION: Dynamic at edge for multi-tenant apps
```

### Cloudflare Workers — Edge Authentication

```javascript
// Verify JWT at the edge — protect origin from unauthenticated requests
// worker.js

const PUBLIC_PATHS = ["/", "/login", "/signup", "/public"];

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const path = url.pathname;

    // Allow public paths
    if (PUBLIC_PATHS.some((p) => path.startsWith(p))) {
      return fetch(request);
    }

    // Require auth for everything else
    const token = extractToken(request);
    if (!token) {
      return new Response("Unauthorized", {
        status: 401,
        headers: { "WWW-Authenticate": "Bearer" },
      });
    }

    // Verify JWT using Web Crypto API (available at edge)
    const isValid = await verifyJWT(token, env.JWT_SECRET);
    if (!isValid) {
      return new Response("Forbidden", { status: 403 });
    }

    // Forward to origin with verified user info
    const newRequest = new Request(request, {
      headers: {
        ...Object.fromEntries(request.headers),
        "X-User-Verified": "true",
        "X-User-Id": extractUserId(token),
      },
    });

    return fetch(newRequest);
  },
};

function extractToken(request) {
  const auth = request.headers.get("Authorization");
  const cookie = parseCookies(request).get("session");
  return auth?.startsWith("Bearer ") ? auth.slice(7) : cookie;
}

async function verifyJWT(token, secret) {
  try {
    const [header, payload, signature] = token.split(".");
    const key = await crypto.subtle.importKey(
      "raw",
      new TextEncoder().encode(secret),
      { name: "HMAC", hash: "SHA-256" },
      false,
      ["verify"],
    );
    return crypto.subtle.verify(
      "HMAC",
      key,
      base64UrlDecode(signature),
      new TextEncoder().encode(`${header}.${payload}`),
    );
  } catch {
    return false;
  }
}
```

### Edge A/B Testing

```javascript
// Assign A/B variants at the edge
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // Only run A/B test on the checkout page
    if (!url.pathname.startsWith("/checkout")) {
      return fetch(request);
    }

    // Get or assign variant
    const cookie = parseCookies(request);
    let variant =
      cookie.get("ab-checkout") ?? assignVariant(["control", "treatment"]);

    // Fetch the appropriate variant
    url.searchParams.set("variant", variant);
    const response = await fetch(new Request(url.toString(), request));

    // Set variant cookie for consistency
    const headers = new Headers(response.headers);
    if (!cookie.get("ab-checkout")) {
      headers.set(
        "Set-Cookie",
        `ab-checkout=${variant}; Max-Age=604800; Path=/; SameSite=Lax`,
      );
    }

    return new Response(response.body, { status: response.status, headers });
  },
};

function assignVariant(variants) {
  return variants[Math.floor(Math.random() * variants.length)];
}
```

---

## 7. Image Optimization via CDN

CDNs can transform images on-the-fly based on URL parameters.

### CDN Image Transformation

```html
<!-- Original image at CDN -->
<img src="https://cdn.example.com/images/hero.jpg" />

<!-- Resized: CDN transforms on first request, caches the result -->
<img src="https://cdn.example.com/images/hero.jpg?w=800&h=400&fit=cover" />

<!-- WebP format (browser that supports it gets WebP, others get JPEG) -->
<img src="https://cdn.example.com/images/hero.jpg?format=auto&w=800" />

<!-- Cloudflare Images URL format -->
<img
  src="https://example.com/cdn-cgi/image/width=800,format=auto/images/hero.jpg"
/>

<!-- Imgix URL format -->
<img src="https://example.imgix.net/hero.jpg?w=800&auto=format&fit=max" />
```

### Responsive Images with CDN

```html
<picture>
  <!-- WebP for modern browsers, JPEG fallback -->
  <source
    srcset="
      https://cdn.example.com/hero.jpg?w=400&format=webp   400w,
      https://cdn.example.com/hero.jpg?w=800&format=webp   800w,
      https://cdn.example.com/hero.jpg?w=1200&format=webp 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1000px) 800px, 1200px"
    type="image/webp"
  />
  <source
    srcset="
      https://cdn.example.com/hero.jpg?w=400   400w,
      https://cdn.example.com/hero.jpg?w=800   800w,
      https://cdn.example.com/hero.jpg?w=1200 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1000px) 800px, 1200px"
    type="image/jpeg"
  />
  <img
    src="https://cdn.example.com/hero.jpg?w=800"
    alt="Hero image"
    width="800"
    height="400"
  />
</picture>
```

### Image CDN Configuration

```javascript
// Cloudflare Image Resizing configuration
// wrangler.toml
// [images]
// bind = "IMAGES"

// Worker: generate image URL
function imageUrl(path, options) {
  const params = Object.entries({
    width: options.width,
    height: options.height,
    fit: options.fit ?? "cover",
    format: options.format ?? "auto", // auto: WebP if supported
    quality: options.quality ?? 85,
  })
    .filter(([_, v]) => v !== undefined)
    .map(([k, v]) => `${k}=${v}`)
    .join(",");

  return `https://example.com/cdn-cgi/image/${params}${path}`;
}

// Usage:
<img src={imageUrl("/products/shoes.jpg", { width: 400, height: 400 })} />;
```

---

## 8. CDN and Security

### DDoS Protection

```
CDN as DDoS shield:
  1. Attacker sends 1,000,000 requests/second
  2. CDN absorbs: cache HITs served from edge (no origin hit)
  3. Cache MISSes: CDN rate limits / challenge / block before reaching origin
  4. Origin: sees 1% of attack traffic (cache hit rate protection)

Tools:
  - Rate limiting by IP: Cloudflare Rules
  - Bot challenges: Cloudflare Turnstile, hCaptcha at edge
  - IP reputation: block known bad actors at edge
  - Geo-blocking: restrict access by country at edge
```

### TLS Termination at Edge

```
WITHOUT CDN:
  User ←──[TLS handshake: 150ms]──→ Origin (far away)
  TLS negotiation adds to TTFB

WITH CDN:
  User ←──[TLS handshake: 8ms]──→ Edge (nearby)
       Edge ←──[pre-warmed TLS]──→ Origin

  CDN maintains persistent TLS to origin (no handshake per user request)
  User's TLS terminates at nearby edge = fast
```

### Security Headers via CDN

```javascript
// Add security headers at edge (Cloudflare Worker)
// Avoids modifying every origin response

export default {
  async fetch(request) {
    const response = await fetch(request);
    const headers = new Headers(response.headers);

    // Security headers
    headers.set(
      "Strict-Transport-Security",
      "max-age=31536000; includeSubDomains; preload",
    );
    headers.set("X-Content-Type-Options", "nosniff");
    headers.set("X-Frame-Options", "DENY");
    headers.set("Referrer-Policy", "strict-origin-when-cross-origin");
    headers.set(
      "Permissions-Policy",
      "geolocation=(), microphone=(), camera=()",
    );
    headers.set("Content-Security-Policy", buildCSP());

    return new Response(response.body, { status: response.status, headers });
  },
};

function buildCSP() {
  return [
    "default-src 'self'",
    "script-src 'self' 'nonce-{NONCE}' cdn.example.com",
    "style-src  'self' 'unsafe-inline' fonts.googleapis.com",
    "img-src    'self' data: cdn.example.com",
    "font-src   'self' fonts.gstatic.com",
    "connect-src 'self' api.example.com",
  ].join("; ");
}
```

---

## 9. CDN Performance Tuning

### HTTP/2 and HTTP/3 Push

```
HTTP/2 PUSH (deprecated, use preload instead):
  Server pushes resources before browser requests them
  Modern approach: <link rel="preload"> in HTML
  CDN delivers preloaded resources from cache

HTTP/3 (QUIC):
  UDP-based protocol
  Better performance on lossy networks (mobile)
  Connection migration (handoff between WiFi and cellular)
  Cloudflare: HTTP/3 enabled by default
  Benefit: 10-30% faster initial connections on bad networks
```

### Connection Optimization

```html
<!-- Preconnect to CDN: establish TLS connection early -->
<link rel="preconnect" href="https://cdn.example.com" crossorigin />

<!-- DNS prefetch as fallback -->
<link rel="dns-prefetch" href="https://cdn.example.com" />

<!-- Preload critical assets from CDN -->
<link
  rel="preload"
  href="https://cdn.example.com/fonts/inter.woff2"
  as="font"
  crossorigin
/>
<link
  rel="preload"
  href="https://cdn.example.com/assets/app.abc123.js"
  as="script"
/>
```

### CDN and Core Web Vitals

```
LCP optimization via CDN:
  LCP image (hero): CDN delivers from edge (8ms vs 180ms TTFB)
  CDN image transformation: exact size for device → no scaling work
  CDN WebP/AVIF: smaller format → faster download
  Result: LCP 0.8s instead of 3.0s

FID/INP optimization via CDN:
  JS bundle: CDN delivers fast → parses sooner → interactive sooner
  CDN HTTP/3: fewer round trips for JS chunks → faster TTI

CLS optimization via CDN:
  Image dimensions in HTML: reserve space before image loads
  CDN-served images: faster load → less time for CLS to occur
  Fonts via CDN: faster → less FOUT (Flash of Unstyled Text)
```

---

## 10. Multi-CDN Architecture

Using multiple CDNs simultaneously for maximum reliability and performance:

```
SINGLE CDN (typical):
  → Single point of failure (CDN outage = outage for you)
  → One CDN may underperform in specific regions

MULTI-CDN (advanced):
  → Route users to best CDN per region
  → Failover: if one CDN is slow/down, switch to another
  → Performance: use CDN A in Asia, CDN B in Europe

IMPLEMENTATION OPTIONS:

1. DNS-based routing (simplest):
   DNS provider (Route53, Cloudflare) routes to different CDNs based on user's geography

2. Anycast routing:
   Multiple CDNs share the same IP block via BGP
   Network routing selects best CDN automatically

3. Client-side switching:
   Monitor CDN performance from client
   Switch to backup CDN URL if primary is slow
```

```javascript
// Client-side CDN failover (simplified)
class CDNRouter {
  #primary   = 'https://cdn1.example.com';
  #secondary = 'https://cdn2.example.com';
  #currentCDN: string;
  #failoverUntil = 0;

  constructor() {
    this.#currentCDN = this.#primary;
  }

  async fetch(path, options = {}) {
    const isFailingOver = Date.now() < this.#failoverUntil;
    const baseUrl = isFailingOver ? this.#secondary : this.#currentCDN;

    try {
      const controller = new AbortController();
      const timeout    = setTimeout(() => controller.abort(), 3000); // 3s timeout

      const response = await fetch(`${baseUrl}${path}`, {
        ...options,
        signal: controller.signal,
      });

      clearTimeout(timeout);

      if (!response.ok && !isFailingOver) {
        this.#triggerFailover();
        return this.fetch(path, options); // retry with secondary
      }

      return response;
    } catch (err) {
      if (!isFailingOver) {
        this.#triggerFailover();
        return this.fetch(path, options);
      }
      throw err;
    }
  }

  #triggerFailover() {
    console.warn('[CDN] Failing over to secondary CDN');
    this.#failoverUntil = Date.now() + 5 * 60_000; // fail over for 5 minutes
    analytics.track('cdn_failover', { from: this.#primary, to: this.#secondary });
  }
}

export const cdn = new CDNRouter();
```

---

## 11. Measuring CDN Performance

### Cache Hit Rate

```javascript
// Server-side: track cache outcomes via CDN headers
function logCacheOutcome(request, response) {
  const cfStatus = response.headers.get("CF-Cache-Status"); // Cloudflare
  const xCache = response.headers.get("X-Cache"); // Generic (HIT/MISS)
  const xServedBy = response.headers.get("X-Served-By"); // Fastly

  const isHit = cfStatus === "HIT" || xCache?.startsWith("HIT");

  metrics.track("cdn_cache", {
    url: request.url,
    status: cfStatus ?? xCache ?? "unknown",
    is_hit: isHit,
    ttfb_ms: performance.timing.responseStart - performance.timing.requestStart,
  });
}
```

### Real User Monitoring (RUM) for CDN Performance

```javascript
// Measure actual latency from user's location to CDN
const timings = performance.getEntriesByType("resource");

timings.forEach((entry) => {
  if (!entry.initiatorType === "script" && !entry.initiatorType === "img")
    return;

  const ttfb = entry.responseStart - entry.requestStart; // time to first byte from CDN
  const isCdnResource = entry.name.includes("cdn.example.com");

  if (isCdnResource) {
    analytics.track("cdn_resource_ttfb", {
      url: entry.name,
      ttfb_ms: ttfb,
      transfer_size: entry.transferSize,
      from_cache: entry.transferSize === 0,
    });
  }
});
```

### CDN Cache Hit Rate Dashboard

```
Target metrics:
  Overall cache hit rate: > 90%
  Static assets (JS/CSS): > 99%
  Images: > 85%
  API responses: > 70% (public endpoints)
  HTML pages: > 80%

Warning signals:
  Static asset cache hit rate < 95%:
    → Check content hashing in filenames
    → Check Cache-Control headers on static assets

  API cache hit rate < 50%:
    → Check Vary headers (Vary: Cookie kills CDN caching)
    → Check if responses are marked private
    → Check query parameter normalization

  TTFB > 100ms despite CDN:
    → Check origin shield configuration
    → Check CDN PoP coverage for your user regions
```

---

## 12. Good Practices

### ✅ Use content hashing for all static assets

```javascript
// Vite/Webpack: content hash in filenames
output: {
  filename: "[name].[contenthash:8].js";
}
// → app.a1b2c3d4.js
// CDN caches with: Cache-Control: public, max-age=31536000, immutable
// No invalidation needed — new content = new URL
```

### ✅ Configure origin shield

```
Origin shield routes all CDN misses through one "shield" node
before hitting origin. Dramatically reduces origin traffic.

Cloudflare: Tiered Cache (Free+)
Fastly: Shielding
Akamai: SureRoute
CloudFront: Origin Shield (extra cost)

Benefit:
  Without shield: 310 PoPs × (requests/PoP) = origin requests
  With shield: 1 shield node fetches, all 310 serve from cache
```

### ✅ Deploy assets before HTML

```
Deployment order matters:
1. Upload new JS/CSS/image assets to CDN (new content-hashed URLs)
2. Deploy new HTML (which references the new asset URLs)

If reversed:
  New HTML deployed first → users get HTML referencing assets that don't exist yet
  → 404 errors for JS/CSS → broken page

Content-hashed assets can be deployed days before needed — zero impact
```

---

## 13. Bad Practices

### ❌ Caching POST responses at CDN

```http
# ❌ POST requests are not idempotent — should not be cached
POST /api/orders HTTP/1.1
Cache-Control: public, max-age=3600
# This would serve the same order creation response to different users

# ✅ POST should always be uncacheable (CDN respects this by default)
# Explicit: Cache-Control: no-store
```

### ❌ Vary: Cookie on publicly cacheable responses

```http
# ❌ Vary: Cookie makes CDN create separate entry per cookie value
# Effectively disables CDN caching (every user has different cookies)
Vary: Cookie, Accept-Encoding
Cache-Control: public, max-age=3600
# → no shared CDN cache entry for this URL

# ✅ Private + no-cache for cookie-dependent responses
# Or: Move user-specific parts to client-side rendering
Cache-Control: private, max-age=30
```

### ❌ Not cleaning up stale edge cache after deploys

```
❌ Deploy new app version: forget to purge HTML pages
   Users receive: cached HTML from CDN (old version)
   Links point to: new asset URLs (deployed but HTML says old URLs)

Result: users see old app or experience 404 errors for updated scripts

✅ Always include CDN purge in deployment pipeline:
  1. Build: new content-hashed assets
  2. Deploy assets: uploaded to CDN/S3/storage
  3. Deploy HTML: to origin server
  4. Purge CDN: purge HTML pages and sitemap
  5. Verify: check CF-Cache-Status: MISS on first request
```

---

## 14. Common Mistakes

### Mistake 1 — Forgetting the `Age` header reveals CDN cache age

```http
# Age: seconds since this response was first cached by CDN
Cache-Control: max-age=3600
Age: 2800

# This response has been in CDN cache for 2800 seconds
# Remaining freshness: 3600 - 2800 = 800 seconds (13 minutes)

# Debug tip: check Age header to understand CDN cache state
# Age=0: just fetched from origin
# Age=3000 when max-age=3600: nearing expiry, will refresh soon
```

### Mistake 2 — CDN URL configuration with CNAME flattening issues

```dns
# CNAME records for CDN:
www.example.com. CNAME → cdn.provider.com.
# CDN serves with SSL for example.com

# Problem: CNAME at root (@) is invalid DNS
# example.com. CNAME → cdn.provider.com.  ← INVALID

# Solution: CNAME flattening (Cloudflare) or A record
# Cloudflare: Add example.com as CNAME — Cloudflare flattens to A record
# AWS: Use Alias records (Route 53) for root domain
```

### Mistake 3 — Not configuring proper TLS for origin to CDN

```
# CDN → Origin communication must also be HTTPS (encrypted)
# Some setups: CDN terminates SSL, then uses HTTP to origin (insecure!)

# CDN to origin modes:
# HTTP: no encryption origin → CDN (insecure if sensitive data)
# HTTPS: encrypted but may not verify origin certificate (still some risk)
# HTTPS + Verify: encrypted + certificate verified (most secure)
# mTLS: mutual TLS, CDN presents client certificate to origin (most secure)

# Cloudflare SSL modes:
# Off → HTTP everywhere (don't use)
# Flexible → CDN to browser: HTTPS, CDN to origin: HTTP (avoid)
# Full → HTTPS everywhere, no cert verification
# Full (Strict) → HTTPS + verified cert (use this)
```

---

## 15. Interview-Level Explanation

> **"How does a CDN work? How do you design a CDN caching strategy for a web application?"**

**Strong answer:**

> "A CDN distributes cached copies of content across geographically distributed edge nodes. When a user requests a resource, their DNS resolves to the nearest edge node — a server potentially 5-50 miles away rather than a data center thousands of miles away. If the edge has the resource cached, it serves it immediately. If not, it fetches from origin, caches it, and serves it. Subsequent users at the same edge get instant cache hits. The primary benefit is latency reduction, not bandwidth — though the bandwidth savings and origin load reduction are significant secondary benefits.
>
> A CDN caching strategy starts with categorizing resources by mutability. Static assets with content hashes in their filenames — `app.abc123.js` — get `Cache-Control: public, max-age=31536000, immutable`. The URL changes when content changes, so the cached file is always correct. You never need to purge these. HTML pages reference those versioned URLs and get `Cache-Control: no-cache` with an ETag — meaning the CDN revalidates with origin on every request, but if unchanged serves a 304 with no body. Public API responses get `s-maxage` to set the CDN TTL separately from the browser TTL — `max-age=300, s-maxage=86400` means the browser considers it fresh for 5 minutes but the CDN holds it for 24 hours. User-specific responses get `Cache-Control: private` to prevent CDN caching entirely.
>
> `Vary` headers require special care. `Vary: Accept-Encoding` is universal and necessary — different encoding formats are different content. `Vary: Cookie` is nearly always a mistake for CDN-cached resources — it means every unique set of cookies gets a separate cache entry, effectively eliminating CDN caching.
>
> Cache invalidation for mutable content has two main approaches: short TTL with stale-while-revalidate (users always get fast cached responses, at most slightly stale), or long TTL with explicit CDN purge on change (requires a purge API call when content changes). Content-hashed assets bypass the problem entirely. For HTML, I keep TTL short and rely on ETag revalidation for efficiency.
>
> Edge functions extend CDN value: JWT verification at edge protects origin from unauthenticated traffic without a round trip, A/B testing at edge avoids origin involvement, and security headers can be added at edge so every response gets them without modifying origin code."

---

## 16. Exercises

### Exercise 1 — CDN configuration audit

You're setting up a CDN for an e-commerce site. For each resource type, specify the appropriate Cache-Control headers and whether to purge on deploy:

```
Resources:
a) /assets/main.a1b2c3d4.js  (content-hashed JS bundle)
b) /assets/hero.jpg           (unversioned product hero image)
c) /                          (homepage HTML)
d) /api/products              (public product catalog, changes hourly)
e) /api/user/cart             (user's cart, personal)
f) /api/auth/login            (POST endpoint)
g) /robots.txt                (rarely changes)
h) /sitemap.xml               (rebuilds on new content)
```

<details>
<summary>Solution</summary>

```
a) /assets/main.a1b2c3d4.js
   Cache-Control: public, max-age=31536000, immutable
   Purge on deploy: NO (URL changes with new content hash)
   Rationale: content-hashed → URL is the version → infinite cache is safe

b) /assets/hero.jpg
   Cache-Control: public, max-age=86400, stale-while-revalidate=3600
   Purge on deploy: YES (URL unchanged, content might change)
   Rationale: images change occasionally; SWR means fast response + eventual freshness
   Better: add version to URL → hero.v2.jpg → immutable caching

c) / (homepage HTML)
   Cache-Control: public, no-cache, s-maxage=60, stale-while-revalidate=300
   CDN-Cache-Control: max-age=60, stale-while-revalidate=300
   ETag: [hash of page content]
   Purge on deploy: YES (new HTML references new asset URLs)
   Rationale: must be fresh after deploy; CDN serves stale while revalidating in background

d) /api/products
   Cache-Control: public, s-maxage=3600, stale-while-revalidate=600, max-age=60
   Purge on deploy: Trigger purge when product data changes (webhook/CI)
   Cache-Tag: products (for tag-based purge)
   Rationale: CDN caches 1 hour; browser 1 min (gets fresh-ish data on page loads)

e) /api/user/cart
   Cache-Control: private, no-cache
   Purge on deploy: N/A (not CDN-cached)
   Rationale: personal data → never CDN cache; no-cache = validate every time

f) /api/auth/login (POST)
   Cache-Control: no-store
   Purge on deploy: N/A (POST, not cached)
   Rationale: auth endpoint, mutation → never cache

g) /robots.txt
   Cache-Control: public, max-age=86400, s-maxage=604800
   Purge on deploy: Only if robots.txt changes
   Rationale: rarely changes; CDN caches 1 week; browser 1 day

h) /sitemap.xml
   Cache-Control: public, max-age=3600, s-maxage=86400
   Purge on deploy: YES (whenever new content is added)
   Cache-Tag: sitemap
   Rationale: changes with new content; CDN caches 1 day between purges
```

</details>

---

## 🔗 Related Topics

- [`caching/01-http-caching.md`](./01-http-caching.md) — HTTP cache headers that CDN respects
- [`caching/02-service-worker-cache.md`](./02-service-worker-cache.md) — SW cache as client-side CDN
- [`performance/08-bundle-optimization.md`](../performance/08-bundle-optimization.md) — What to put on the CDN
- [`browser-internals/08-critical-rendering-path.md`](../browser-internals/08-critical-rendering-path.md) — CDN impact on CRP
- [`networking/01-http-protocols.md`](../networking/01-http-protocols.md) — HTTP/2, HTTP/3 over CDN

---

<div align="center">

**`caching/` section complete!** 🎉

All 5 caching files done:
`01-http-caching.md` · `02-service-worker-cache.md` · `03-memory-caching.md` · `04-data-caching.md` · **`05-cdn-strategies.md`** ✓

**Next section:** [`networking/`](../networking/) →

</div>
