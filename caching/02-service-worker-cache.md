# 02 — Service Worker Cache

> **"The Service Worker cache is the HTTP cache with superpowers. The HTTP cache follows the server's instructions. The Service Worker cache follows yours. Every request goes through your code first — you decide what to cache, how long to keep it, and what to serve when the network fails."**

Service Workers intercept network requests and give JavaScript full control over caching behavior. Unlike HTTP caching which is driven by response headers you may not control, the Cache API lets you implement any caching strategy programmatically. This document covers the five core Service Worker caching strategies, cache versioning and cleanup, background sync, cache storage limits, and the practical patterns that power reliable offline-capable web applications.

---

## 📚 Table of Contents

1. [Service Worker Architecture](#1-service-worker-architecture)
2. [The Cache API](#2-the-cache-api)
3. [Strategy 1 — Cache First (Offline First)](#3-strategy-1--cache-first-offline-first)
4. [Strategy 2 — Network First](#4-strategy-2--network-first)
5. [Strategy 3 — Stale While Revalidate](#5-strategy-3--stale-while-revalidate)
6. [Strategy 4 — Cache Only](#6-strategy-4--cache-only)
7. [Strategy 5 — Network Only](#7-strategy-5--network-only)
8. [Strategy Routing — The Right Strategy Per Request](#8-strategy-routing--the-right-strategy-per-request)
9. [Cache Versioning and Cleanup](#9-cache-versioning-and-cleanup)
10. [Workbox — Production Service Worker Toolkit](#10-workbox--production-service-worker-toolkit)
11. [Background Sync](#11-background-sync)
12. [Cache Storage Limits](#12-cache-storage-limits)
13. [Service Worker Lifecycle](#13-service-worker-lifecycle)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. Service Worker Architecture

```
WITHOUT SERVICE WORKER:
  Page → Browser Cache → Network → Server

WITH SERVICE WORKER:
  Page → [Service Worker JavaScript] → Browser Cache / Network → Server
              ↑
              Your code intercepts here
              You decide: serve from cache? fetch? combine?

SERVICE WORKER PROPERTIES:
  - Runs in a separate thread (no DOM access)
  - Persists across page loads (even when page is closed)
  - Intercepting fetch events: can modify, cache, or return any response
  - HTTPS only (security requirement — except localhost)
  - Shared across all tabs for the same origin
```

### Service Worker Registration

```javascript
// app.js — main page context
if ("serviceWorker" in navigator) {
  window.addEventListener("load", async () => {
    try {
      const registration = await navigator.serviceWorker.register(
        "/sw.js", // path to SW file
        { scope: "/" }, // controls which URLs it intercepts
      );

      // Listen for updates
      registration.addEventListener("updatefound", () => {
        const newWorker = registration.installing;
        newWorker.addEventListener("statechange", () => {
          if (
            newWorker.state === "installed" &&
            navigator.serviceWorker.controller
          ) {
            // New version available — prompt user to refresh
            showUpdateBanner();
          }
        });
      });

      console.log("SW registered:", registration.scope);
    } catch (err) {
      console.error("SW registration failed:", err);
    }
  });
}
```

---

## 2. The Cache API

The Cache API provides a persistent key-value store for Request/Response pairs:

```javascript
// Opening a named cache
const cache = await caches.open("my-cache-v1");

// Storing a response
await cache.put(request, response);

// Storing a URL (auto-fetches and stores)
await cache.add("/assets/app.js");

// Storing multiple URLs
await cache.addAll([
  "/",
  "/offline.html",
  "/assets/app.js",
  "/assets/styles.css",
]);

// Retrieving a cached response
const response = await cache.match(request);
// returns Response or undefined

// Matching with options
const response = await cache.match(request, {
  ignoreSearch: true, // ignore query string in URL matching
  ignoreMethod: true, // match regardless of HTTP method
  ignoreVary: false, // respect Vary header (default)
});

// Listing all keys
const keys = await cache.keys();

// Deleting a specific entry
await cache.delete(request);

// Deleting an entire named cache
await caches.delete("my-cache-v1");

// Listing all cache names
const cacheNames = await caches.keys();
```

---

## 3. Strategy 1 — Cache First (Offline First)

Serve from cache if available, fall back to network. Best for assets that rarely change.

```javascript
// sw.js
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) {
    return cached; // instant — no network
  }

  // Not in cache: fetch from network and store
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open("static-v1");
      cache.put(request, response.clone()); // clone: body can only be consumed once
    }
    return response;
  } catch (err) {
    // Network failed and not in cache: return offline fallback
    return caches.match("/offline.html");
  }
}

// Use for:
// - Static assets (JS, CSS, images) with content hashing
// - Font files
// - App shell (HTML skeleton)
// - Rarely-changing content

// Trade-off:
// + Instant load for cached resources
// + Works offline
// - Stale content until cache is invalidated/versioned
```

---

## 4. Strategy 2 — Network First

Try the network, fall back to cache if the network fails. Best for frequently-updated content where freshness matters.

```javascript
async function networkFirst(
  request,
  cacheName = "dynamic-v1",
  timeoutMs = 3000,
) {
  const cache = await caches.open(cacheName);

  try {
    // Fetch with timeout
    const response = await Promise.race([
      fetch(request),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error("Network timeout")), timeoutMs),
      ),
    ]);

    // Success: update cache and return fresh response
    if (response.ok) {
      cache.put(request, response.clone()); // update in background
    }
    return response;
  } catch (err) {
    // Network failed or timed out: serve from cache
    const cached = await cache.match(request);
    if (cached) return cached;

    // Neither network nor cache: offline fallback
    if (request.headers.get("Accept")?.includes("text/html")) {
      return caches.match("/offline.html");
    }

    throw err; // propagate non-HTML request failures
  }
}

// Use for:
// - API responses where freshness is important
// - HTML pages
// - User-specific data with offline fallback
// - Content that changes frequently

// Trade-off:
// + Always fresh when online
// + Falls back gracefully when offline
// - Slower than cache first (waits for network)
// - Timeout adds complexity
```

---

## 5. Strategy 3 — Stale While Revalidate

Serve from cache immediately, update the cache in the background. Best for content where instant response is more important than perfect freshness.

```javascript
async function staleWhileRevalidate(request, cacheName = "api-v1") {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);

  // Background revalidation (fire and forget)
  const revalidate = fetch(request)
    .then((response) => {
      if (response.ok) {
        cache.put(request, response.clone());
      }
      return response;
    })
    .catch(() => {}); // silent fail — we already have cached version

  // If cached: return immediately + revalidate in background
  if (cached) {
    return cached;
  }

  // No cache: must wait for network
  return revalidate;
}

// Use for:
// - News/blog content
// - Social feeds
// - Product listings
// - Any content where "slightly stale" is acceptable
// - High-traffic endpoints where cache hit rate matters

// Trade-off:
// + Always instant response (from cache)
// + Eventually consistent (background updates)
// - User may briefly see stale content
// - No guarantee of when update occurs
```

---

## 6. Strategy 4 — Cache Only

Serve only from cache. Never go to the network. Used for pre-cached resources.

```javascript
async function cacheOnly(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  throw new Error(`Cache-only: ${request.url} not in cache`);
}

// Use for:
// - App shell resources pre-cached during install
// - Resources known to be pre-loaded
// - Offline mode toggle (when explicitly offline)

// Rarely used directly — usually part of a larger routing strategy
```

---

## 7. Strategy 5 — Network Only

Always go to the network, never cache. The default browser behavior.

```javascript
async function networkOnly(request) {
  return fetch(request); // passthrough
}

// Use for:
// - Authenticated API requests with no offline benefit
// - Analytics pings (must reach server)
// - POST/PUT/DELETE mutations
// - Requests that must always be fresh (real-time pricing, live inventory)
```

---

## 8. Strategy Routing — The Right Strategy Per Request

A production Service Worker routes different requests to different strategies:

```javascript
// sw.js — complete routing setup

const STATIC_CACHE = "static-v2";
const DYNAMIC_CACHE = "dynamic-v1";
const API_CACHE = "api-v1";

self.addEventListener("fetch", (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // Skip non-GET and non-same-origin requests (mostly)
  if (request.method !== "GET") return;
  if (!url.origin.startsWith("https://")) return; // allow only HTTPS

  // 1. Static assets (JS/CSS/fonts): Cache First (immutable content hash)
  if (isStaticAsset(url)) {
    event.respondWith(cacheFirst(request, STATIC_CACHE));
    return;
  }

  // 2. Images: Stale While Revalidate (freshness not critical)
  if (isImage(url)) {
    event.respondWith(staleWhileRevalidate(request, "images-v1"));
    return;
  }

  // 3. API: varies by endpoint
  if (url.pathname.startsWith("/api/")) {
    // Public catalog: stale-while-revalidate (fast + eventually fresh)
    if (
      url.pathname.startsWith("/api/products") ||
      url.pathname.startsWith("/api/categories")
    ) {
      event.respondWith(staleWhileRevalidate(request, API_CACHE));
      return;
    }

    // Auth endpoints: network only (never cache tokens/sessions)
    if (url.pathname.startsWith("/api/auth/")) {
      return; // fallthrough to default browser behavior
    }

    // User-specific: network first with fallback
    if (url.pathname.startsWith("/api/user/")) {
      event.respondWith(networkFirst(request, DYNAMIC_CACHE, 3000));
      return;
    }
  }

  // 4. HTML navigation: network first (fresh HTML references versioned assets)
  if (request.headers.get("Accept")?.includes("text/html")) {
    event.respondWith(
      networkFirst(request, DYNAMIC_CACHE, 5000).catch(() =>
        caches.match("/offline.html"),
      ),
    );
    return;
  }

  // Default: network only
});

// Route matchers
function isStaticAsset(url) {
  return (
    /\.(js|css|woff2?|ttf|otf)$/.test(url.pathname) ||
    url.pathname.startsWith("/assets/") ||
    url.pathname.startsWith("/_next/static/")
  );
}

function isImage(url) {
  return /\.(png|jpg|jpeg|gif|webp|avif|svg|ico)$/.test(url.pathname);
}
```

---

## 9. Cache Versioning and Cleanup

Cache names must be versioned; old caches must be cleaned up on SW activation.

```javascript
const CACHES = {
  static: "static-v3", // increment on new deployment
  dynamic: "dynamic-v1",
  api: "api-v1",
  images: "images-v1",
};

const ALL_CACHES = Object.values(CACHES);

// INSTALL: pre-cache the app shell
self.addEventListener("install", (event) => {
  event.waitUntil(
    (async () => {
      const cache = await caches.open(CACHES.static);

      // Pre-cache critical resources
      // These URLs are updated by the build tool with correct hashes
      await cache.addAll([
        "/",
        "/offline.html",
        "/assets/app.abc123.js",
        "/assets/styles.def456.css",
        "/assets/fonts/inter.woff2",
      ]);

      // Immediately activate (don't wait for existing tab to close)
      await self.skipWaiting();
    })(),
  );
});

// ACTIVATE: clean up old caches from previous versions
self.addEventListener("activate", (event) => {
  event.waitUntil(
    (async () => {
      const cacheNames = await caches.keys();

      await Promise.all(
        cacheNames
          .filter((name) => !ALL_CACHES.includes(name)) // old cache name
          .map((name) => {
            console.log(`[SW] Deleting old cache: ${name}`);
            return caches.delete(name);
          }),
      );

      // Take control of all clients immediately
      await self.clients.claim();
    })(),
  );
});

// STRATEGY: Cache-first with max-entry and max-age limits
async function cacheFirstWithLimit(
  request,
  cacheName,
  { maxEntries = 50, maxAgeSeconds = 86400 } = {},
) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);

  if (cached) {
    // Check if cached response is still within max age
    const cachedDate = cached.headers.get("sw-cache-date");
    if (cachedDate) {
      const age = (Date.now() - Number(cachedDate)) / 1000;
      if (age < maxAgeSeconds) return cached;
      // Expired: fetch fresh
    } else {
      return cached; // no date header: serve as-is
    }
  }

  const response = await fetch(request);
  if (!response.ok) return response;

  // Add custom cache date header before storing
  const responseToCache = addCacheDateHeader(response.clone());
  await cache.put(request, responseToCache);

  // Enforce max entries limit: delete oldest if exceeded
  await enforceMaxEntries(cache, maxEntries);

  return response;
}

async function enforceMaxEntries(cache, maxEntries) {
  const keys = await cache.keys();
  if (keys.length > maxEntries) {
    // Delete oldest entries (first in array = oldest by insertion order)
    const deleteCount = keys.length - maxEntries;
    await Promise.all(keys.slice(0, deleteCount).map((k) => cache.delete(k)));
  }
}

function addCacheDateHeader(response) {
  const headers = new Headers(response.headers);
  headers.set("sw-cache-date", String(Date.now()));
  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers,
  });
}
```

---

## 10. Workbox — Production Service Worker Toolkit

Workbox is Google's library for Service Worker development. It implements all caching strategies, handles versioning, and integrates with build tools.

```javascript
// sw.js with Workbox
import { precacheAndRoute, cleanupOutdatedCaches } from "workbox-precaching";
import { registerRoute, NavigationRoute, Route } from "workbox-routing";
import {
  CacheFirst,
  NetworkFirst,
  StaleWhileRevalidate,
  NetworkOnly,
} from "workbox-strategies";
import { ExpirationPlugin } from "workbox-expiration";
import { CacheableResponsePlugin } from "workbox-cacheable-response";

// Pre-cache: automatically updated by Workbox's build plugin
// (list of URLs with content hashes — generated at build time)
cleanupOutdatedCaches();
precacheAndRoute(self.__WB_MANIFEST); // injected by build tool

// Static assets: Cache First with 1-year expiration
registerRoute(
  ({ request }) =>
    request.destination === "script" || request.destination === "style",
  new CacheFirst({
    cacheName: "static-assets",
    plugins: [
      new ExpirationPlugin({
        maxAgeSeconds: 365 * 24 * 60 * 60,
        maxEntries: 100,
      }),
      new CacheableResponsePlugin({ statuses: [0, 200] }),
    ],
  }),
);

// Images: Stale While Revalidate with 30-day expiration
registerRoute(
  ({ request }) => request.destination === "image",
  new StaleWhileRevalidate({
    cacheName: "images",
    plugins: [
      new ExpirationPlugin({
        maxAgeSeconds: 30 * 24 * 60 * 60,
        maxEntries: 200,
      }),
      new CacheableResponsePlugin({ statuses: [0, 200] }),
    ],
  }),
);

// API: Stale While Revalidate for public data
registerRoute(
  ({ url }) => url.pathname.startsWith("/api/products"),
  new StaleWhileRevalidate({
    cacheName: "api-products",
    plugins: [
      new ExpirationPlugin({ maxAgeSeconds: 5 * 60, maxEntries: 50 }), // 5 min
      new CacheableResponsePlugin({ statuses: [200] }),
    ],
  }),
);

// Navigation (HTML): Network First with offline fallback
registerRoute(
  new NavigationRoute(
    new NetworkFirst({
      cacheName: "pages",
      plugins: [
        new ExpirationPlugin({ maxEntries: 20 }),
        new CacheableResponsePlugin({ statuses: [200] }),
      ],
    }),
    {
      allowlist: [/^\//],
      denylist: [/\/api\//],
    },
  ),
);
```

### Workbox Build Integration (Vite)

```javascript
// vite.config.ts
import { VitePWA } from "vite-plugin-pwa";

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: "autoUpdate",
      workbox: {
        globPatterns: ["**/*.{js,css,html,ico,png,svg,woff2}"],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\.example\.com\/products/,
            handler: "StaleWhileRevalidate",
            options: {
              cacheName: "api-products",
              expiration: { maxEntries: 50, maxAgeSeconds: 300 },
            },
          },
          {
            urlPattern: /\.(?:png|jpg|jpeg|webp|avif)$/,
            handler: "CacheFirst",
            options: {
              cacheName: "images",
              expiration: { maxEntries: 200, maxAgeSeconds: 2592000 }, // 30d
            },
          },
        ],
      },
    }),
  ],
});
```

---

## 11. Background Sync

Background Sync allows deferred requests to be retried when connectivity is restored:

```javascript
// Queue failed requests for background sync
// sw.js
import { BackgroundSyncPlugin } from "workbox-background-sync";

const bgSyncPlugin = new BackgroundSyncPlugin("formQueue", {
  maxRetentionTime: 24 * 60, // retry up to 24 hours (in minutes)
});

registerRoute(
  ({ url }) => url.pathname === "/api/contact-form",
  new NetworkOnly({ plugins: [bgSyncPlugin] }),
  "POST",
);

// Manual Background Sync (without Workbox):
self.addEventListener("sync", (event) => {
  if (event.tag === "submit-form") {
    event.waitUntil(retrySavedRequests());
  }
});

async function retrySavedRequests() {
  const db = await openDB("sync-queue", 1);
  const items = await db.getAll("requests");

  for (const item of items) {
    try {
      const response = await fetch(item.url, {
        method: item.method,
        headers: item.headers,
        body: item.body,
      });

      if (response.ok) {
        await db.delete("requests", item.id);
      }
    } catch (err) {
      // Still failing — will retry on next sync event
      console.log(`[SW] Retry failed for ${item.url}`, err);
    }
  }
}

// Main thread: register sync when form submitted offline
async function submitForm(data) {
  try {
    return await fetch("/api/contact-form", {
      method: "POST",
      body: JSON.stringify(data),
    });
  } catch {
    // Save for later
    const db = await openDB("sync-queue", 1);
    await db.add("requests", {
      url: "/api/contact-form",
      method: "POST",
      body: JSON.stringify(data),
      savedAt: Date.now(),
    });

    // Register background sync
    const registration = await navigator.serviceWorker.ready;
    await registration.sync.register("submit-form");

    return new Response(JSON.stringify({ queued: true }), { status: 202 });
  }
}
```

---

## 12. Cache Storage Limits

Service Worker caches are subject to browser storage quotas:

```javascript
// Check available storage quota
async function checkStorageQuota() {
  if (!navigator.storage?.estimate) return;

  const { usage, quota } = await navigator.storage.estimate();

  console.log(`Storage used: ${(usage / 1e6).toFixed(2)} MB`);
  console.log(`Storage quota: ${(quota / 1e6).toFixed(2)} MB`);
  console.log(`Usage: ${((usage / quota) * 100).toFixed(1)}%`);

  return { usage, quota, usagePercent: (usage / quota) * 100 };
}

// Request persistent storage (won't be evicted under pressure)
async function requestPersistentStorage() {
  if (navigator.storage?.persist) {
    const granted = await navigator.storage.persist();
    if (granted) {
      console.log("Storage persisted — won't be auto-evicted");
    } else {
      console.log(
        "Persistent storage denied — may be evicted under storage pressure",
      );
    }
  }
}
```

### Browser Quotas (Approximate)

```
CHROME:
  Default quota: up to 60% of available disk space (at most)
  Eviction: LRU when disk is low, can evict ENTIRE ORIGIN's cache
  Persistent: navigator.storage.persist() prevents eviction

SAFARI:
  Default: 50MB per origin (strict)
  Eviction: after 7 days of inactivity the entire cache is cleared
  Note: Apple's Intelligent Tracking Prevention affects SW lifetime

FIREFOX:
  Default quota: 50% of free disk space (at most 2GB)
  Eviction: LRU by origin, warns user when quota approached

PRACTICAL IMPLICATIONS:
  - Don't cache large video files
  - Implement maxEntries / maxAgeSeconds limits
  - Test on Safari specifically — 7-day TTL breaks many offline strategies
  - Request persistent storage for PWAs: navigator.storage.persist()
```

---

## 13. Service Worker Lifecycle

```
INSTALL:
  trigger: new or updated SW detected
  event:   'install'
  use:     pre-cache app shell, set up caches
  gotcha:  if waitUntil() promise rejects: SW fails to install

  self.addEventListener('install', event => {
    event.waitUntil(precache());
    self.skipWaiting(); // activate immediately without waiting for old clients to close
  });

ACTIVATE:
  trigger: after install, when old SW no longer has clients
           (or immediately if skipWaiting() was called)
  event:   'activate'
  use:     delete old caches, take control of clients
  gotcha:  old SW may still be running until all its clients (tabs) are closed

  self.addEventListener('activate', event => {
    event.waitUntil(deleteOldCaches());
    self.clients.claim(); // take control of uncontrolled clients immediately
  });

FETCH:
  trigger: any network request from controlled pages
  event:   'fetch'
  use:     apply caching strategies

WAITING:
  A new SW is installed but waiting for the old one to release control.
  Happens when user has the page open in multiple tabs.
  Solutions:
  - self.skipWaiting() in install: skips waiting
  - User closes all tabs: old SW releases control
  - Programmatic activation: tell all clients to reload

MESSAGE:
  trigger: main thread posts message to SW
  event:   'message'
  use:     skip waiting, clear cache, custom commands
```

### Communicating Between SW and Main Thread

```javascript
// sw.js: listen for messages from main thread
self.addEventListener("message", async (event) => {
  const { type, payload } = event.data;

  switch (type) {
    case "SKIP_WAITING":
      self.skipWaiting();
      break;

    case "CLEAR_CACHE":
      const cacheNames = await caches.keys();
      await Promise.all(cacheNames.map((name) => caches.delete(name)));
      event.ports[0].postMessage({ success: true });
      break;

    case "CACHE_URLS":
      const cache = await caches.open("manual-v1");
      await cache.addAll(payload.urls);
      event.ports[0].postMessage({ cached: payload.urls.length });
      break;
  }
});

// main thread: prompt user to update
function promptUpdate(registration) {
  const updateBanner = document.getElementById("update-banner");
  updateBanner.style.display = "block";

  document.getElementById("update-btn").addEventListener("click", async () => {
    // Tell waiting SW to activate now
    registration.waiting?.postMessage({ type: "SKIP_WAITING" });
    // Reload to use new SW
    window.location.reload();
  });
}

// Listen for SW controller change (new SW activated)
navigator.serviceWorker.addEventListener("controllerchange", () => {
  window.location.reload();
});
```

---

## 14. Good Practices

### ✅ Version cache names and clean up in activate

```javascript
// ✅ Always version caches — clean old ones in activate
const CACHE_VERSION = "v3";
const STATIC_CACHE = `static-${CACHE_VERSION}`;

self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches
      .keys()
      .then((keys) =>
        Promise.all(
          keys
            .filter(
              (k) =>
                !k.startsWith(`static-${CACHE_VERSION}`) &&
                !k.startsWith("api-"),
            )
            .map((k) => caches.delete(k)),
        ),
      ),
  );
});
```

### ✅ Always clone responses before caching

```javascript
// ✅ Body can only be read once — clone before storing
const response = await fetch(request);
cache.put(request, response.clone()); // store clone
return response; // return original to page
```

### ✅ Set expiration limits on dynamic caches

```javascript
// ✅ Prevent unbounded cache growth
new CacheFirst({
  cacheName: "images",
  plugins: [
    new ExpirationPlugin({
      maxEntries: 200, // max 200 images
      maxAgeSeconds: 30 * 86400, // max 30 days old
    }),
  ],
});
```

### ✅ Test offline behavior in DevTools

```
DevTools → Application → Service Workers → Offline checkbox
Tests that your offline fallback works correctly
Also: Application → Storage → Clear storage (test fresh install)
```

---

## 15. Bad Practices

### ❌ Caching authenticated/sensitive requests

```javascript
// ❌ Caching user-specific API responses in SW
event.respondWith(
  staleWhileRevalidate(event.request), // caches /api/user/profile
);
// Problem: multiple users on same device share SW cache
// User A's profile could be served to User B

// ✅ Skip authenticated requests
if (event.request.headers.has("Authorization")) {
  return; // don't cache — let browser handle normally
}
```

### ❌ Missing offline fallback

```javascript
// ❌ No fallback for navigation requests
async function networkFirst(request) {
  try {
    return await fetch(request);
  } catch {
    return new Response("Network error", { status: 503 }); // ← generic error
  }
}

// ✅ Proper offline fallback
async function networkFirst(request) {
  try {
    return await fetch(request);
  } catch {
    const cached = await caches.match(request);
    if (cached) return cached;

    if (request.headers.get("Accept")?.includes("text/html")) {
      return caches.match("/offline.html"); // proper offline page
    }
    return new Response("", { status: 503 });
  }
}
```

### ❌ Caching opaque responses without caution

```javascript
// Opaque response: cross-origin request without CORS
// Status: 0, no headers, can't inspect body
// CDNs may send 400/500 as opaque 0-status responses

// ❌ Blindly caching opaque responses
cache.put(request, response); // could cache a 404 as status 0

// ✅ Only cache opaque responses for specific use cases (icons, fonts)
if (response.type === "opaque") {
  // Only cache if we expect this resource to work
  if (/\.(woff2|ico)$/.test(url.pathname)) {
    cache.put(request, response.clone());
  }
}
// Or: use CacheableResponsePlugin({ statuses: [0, 200] }) from Workbox
```

---

## 16. Common Mistakes

### Mistake 1 — Serving stale HTML that references old assets

```javascript
// ❌ Caching HTML with Cache First + caching assets without versioning
// After deploy: old HTML cached → references old asset URLs
// Old assets cached → correct HTML never reached
// Result: broken app until cache expires

// ✅ HTML: Network First (always fresh HTML with correct asset references)
// Assets: Cache First (content-hashed URLs, immutable)
// Never cache HTML with Cache First
```

### Mistake 2 — Missing event.waitUntil() for async work

```javascript
// ❌ Async work without waitUntil — SW may terminate before it completes
self.addEventListener("activate", (event) => {
  deleteOldCaches(); // not awaited! SW may terminate mid-cleanup
});

// ✅ Always use waitUntil for async work
self.addEventListener("activate", (event) => {
  event.waitUntil(deleteOldCaches()); // SW stays alive until promise resolves
});
```

### Mistake 3 — Not handling install failures gracefully

```javascript
// ❌ A single failed addAll() fails the entire SW installation
self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(
      (cache) => cache.addAll(CRITICAL_ASSETS), // if ANY asset 404s: entire install fails
    ),
  );
});

// ✅ Only include assets you're certain will be available
// Don't addAll() too many resources during install
// Use runtimeCaching for optional resources
```

---

## 17. Interview-Level Explanation

> **"How does Service Worker caching work? What are the different strategies and when do you use each?"**

**Strong answer:**

> "Service Workers sit between the browser and the network, intercepting all fetch events. Inside the fetch event handler, you can implement any caching behavior using the Cache API — a persistent key-value store for Request/Response pairs. This gives you full programmable control over caching decisions that HTTP headers alone can't express.
>
> There are five core strategies. Cache First serves from cache and only goes to the network on a miss — best for immutable content like content-hashed JavaScript and CSS bundles where you want instant load. Network First tries the network and falls back to cache on failure — best for HTML pages and user-specific API data where freshness matters but you want offline resilience. Stale While Revalidate serves cached content immediately while updating the cache in the background — best for product listings, news feeds, and any content where 'slightly stale' is acceptable but instant response is important. Cache Only serves exclusively from pre-cached resources — used for the app shell or during offline mode. Network Only is the default browser behavior — used for auth endpoints, analytics, and mutations.
>
> In production, you route different request types to different strategies. Static assets with content hashes use Cache First with year-long expiration. HTML navigation uses Network First with an offline fallback page. Public API responses use Stale While Revalidate. Auth-related requests are explicitly excluded from SW caching.
>
> Cache versioning is critical: you name caches with version suffixes (static-v3, api-v1) and delete old named caches in the activate event. This ensures old cached resources don't persist after a new SW deploys.
>
> The gotchas: always clone responses before storing (the body stream can only be consumed once). Never cache responses with Authorization headers (multi-user device security). Don't use Cache First for HTML (it blocks users from getting updated asset URLs after deployments). Be careful with opaque responses — you can't tell if they're 200 or 404. And Safari clears all SW caches after 7 days of inactivity, which breaks many offline strategies on iOS without explicit workarounds.
>
> Workbox handles most of this correctly by default and integrates with build tools to generate the precache manifest automatically."

---

## 18. Exercises

### Exercise 1 — Design a caching strategy

For a recipe website with these pages and resources, design a complete Service Worker caching strategy:

- Homepage (shows latest 20 recipes, updates daily)
- Recipe detail pages (mostly static content, rarely changes)
- Recipe images (many, large, rarely changes once added)
- User's saved recipes list (authenticated, personal)
- Search API `/api/search?q=...` (public, real-time)

<details>
<summary>Solution</summary>

```javascript
// Service Worker strategy for recipe website

self.addEventListener("fetch", (event) => {
  const url = new URL(event.request.url);
  const { request } = event;

  if (request.method !== "GET") return;

  // STATIC ASSETS (content-hashed): Cache First, immutable
  if (/\/_next\/static\/|\/assets\//.test(url.pathname)) {
    event.respondWith(cacheFirst(request, "static-v1"));
    return;
  }

  // RECIPE IMAGES: Cache First with long TTL (images don't change once uploaded)
  if (/\/images\/recipes\//.test(url.pathname)) {
    event.respondWith(
      cacheFirstWithLimit(request, "recipe-images", {
        maxEntries: 200,
        maxAgeSeconds: 30 * 86400, // 30 days
      }),
    );
    return;
  }

  // SEARCH API: Network Only (real-time, results must be current)
  if (url.pathname.startsWith("/api/search")) {
    return; // fallthrough: no SW caching
  }

  // USER DATA (authenticated): Network First, no aggressive caching
  if (url.pathname.startsWith("/api/user/")) {
    if (request.headers.has("Authorization")) {
      event.respondWith(networkFirst(request, "user-data", 5000));
    }
    return;
  }

  // RECIPE DETAIL PAGES: Stale While Revalidate (rarely changes, want instant load)
  if (url.pathname.startsWith("/recipes/")) {
    event.respondWith(staleWhileRevalidate(request, "recipe-pages"));
    return;
  }

  // HOMEPAGE: Network First with short TTL (updates daily, want fresh)
  if (url.pathname === "/" || url.pathname === "/index.html") {
    event.respondWith(
      networkFirst(request, "pages", 5000).catch(() =>
        caches.match("/offline.html"),
      ),
    );
    return;
  }

  // HTML pages generally: Network First
  if (request.headers.get("Accept")?.includes("text/html")) {
    event.respondWith(
      networkFirst(request, "pages", 5000).catch(() =>
        caches.match("/offline.html"),
      ),
    );
    return;
  }
});

// Pre-cache on install
self.addEventListener("install", (event) => {
  event.waitUntil(
    caches
      .open("static-v1")
      .then((cache) =>
        cache.addAll([
          "/offline.html",
          // content-hashed assets injected by build tool
        ]),
      )
      .then(() => self.skipWaiting()),
  );
});

// Clean old caches on activate
self.addEventListener("activate", (event) => {
  const CURRENT = [
    "static-v1",
    "recipe-images",
    "recipe-pages",
    "pages",
    "user-data",
  ];
  event.waitUntil(
    caches
      .keys()
      .then((keys) =>
        Promise.all(
          keys.filter((k) => !CURRENT.includes(k)).map((k) => caches.delete(k)),
        ),
      )
      .then(() => self.clients.claim()),
  );
});
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/13-service-workers.md`](../javascript-core/13-service-workers.md) — Service Worker fundamentals
- [`caching/01-http-caching.md`](./01-http-caching.md) — HTTP cache headers that complement SW cache
- [`caching/03-memory-caching.md`](./03-memory-caching.md) — In-memory caching patterns
- [`browser-internals/09-browser-caching.md`](../browser-internals/09-browser-caching.md) — Browser cache internals

---

<div align="center">

**Next:** [`caching/03-memory-caching.md`](./03-memory-caching.md) →

</div>
