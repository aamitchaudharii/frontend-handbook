# 13 — Service Workers

> **"A Service Worker is not a background script. It's a programmable network proxy that sits between your application and the internet — and unlike every other worker, it survives after the browser tab closes."**

Service Workers are the foundation of Progressive Web Apps. They enable offline experiences, background sync, push notifications, and fine-grained caching strategies that no other browser API can match. This document covers the complete lifecycle, every caching strategy, push notifications, background sync, and production patterns for real-world PWAs.

---

## 📚 Table of Contents

1. [What Makes Service Workers Unique](#1-what-makes-service-workers-unique)
2. [The Service Worker Lifecycle](#2-the-service-worker-lifecycle)
3. [Registration](#3-registration)
4. [Install Event — Pre-Caching](#4-install-event--pre-caching)
5. [Activate Event — Cache Cleanup](#5-activate-event--cache-cleanup)
6. [Fetch Event — Network Interception](#6-fetch-event--network-interception)
7. [Caching Strategies](#7-caching-strategies)
8. [Cache API — Deep Dive](#8-cache-api--deep-dive)
9. [Background Sync](#9-background-sync)
10. [Push Notifications](#10-push-notifications)
11. [Service Worker Communication](#11-service-worker-communication)
12. [Updating a Service Worker](#12-updating-a-service-worker)
13. [Service Worker Scope](#13-service-worker-scope)
14. [Debugging Service Workers](#14-debugging-service-workers)
15. [Production Architecture](#15-production-architecture)
16. [Good Practices](#16-good-practices)
17. [Bad Practices](#17-bad-practices)
18. [Common Mistakes](#18-common-mistakes)
19. [Interview-Level Explanation](#19-interview-level-explanation)
20. [Exercises](#20-exercises)

---

## 1. What Makes Service Workers Unique

Service Workers differ from every other JavaScript environment in three fundamental ways:

### 1. They Survive Page Close

```
Normal script/worker:
  Page open → script runs → Page closed → script terminates

Service Worker:
  Page open → SW installs/activates
  Page closed → SW may continue running (handling push events, sync)
  Page opened again → SW is already active, intercepts requests immediately
```

### 2. They Are a Network Proxy

```
Without Service Worker:
  Page → fetch('/api/data') → Internet → Server → Response → Page

With Service Worker:
  Page → fetch('/api/data') → SW intercepts → (decide: cache? network? both?)
                                    ↓
                              Return response (from cache, network, or generated)
```

Every `fetch()` call, every image request, every API call — all pass through the Service Worker's `fetch` event handler. You decide what to return.

### 3. They Require HTTPS

Service Workers only work on:

- `https://` — production
- `http://localhost` — development exception
- `http://127.0.0.1` — development exception

This is a security requirement: a compromised SW could intercept all traffic, so it's only allowed on secure origins.

### 4. They Have Their Own Lifecycle

Unlike normal workers, Service Workers have an explicit install → activate → idle → fetch lifecycle with update mechanics that prevent breaking active users.

---

## 2. The Service Worker Lifecycle

```
                    navigator.serviceWorker.register('./sw.js')
                                    │
                                    ▼
                              PARSED
                         (SW script downloaded
                          and parsed)
                                    │
                                    ▼
                    ┌─────── INSTALLING ──────────┐
                    │  install event fires        │
                    │  waitUntil(promise) holds   │
                    │  If promise rejects →       │
                    │  REDUNDANT                  │
                    └─────────────────────────────┘
                                    │ install complete
                                    ▼
                              INSTALLED
                    (waiting for old SW to release)
                                    │
                                    ▼
                    ┌─────── ACTIVATING ──────────┐
                    │  activate event fires       │
                    │  Old SW is gone             │
                    │  Clean up old caches        │
                    └─────────────────────────────┘
                                    │ activate complete
                                    ▼
                             ACTIVATED
                    (controls pages, handles fetch)
                                    │
                    ┌───────────────┴──────────────────┐
                    ▼                                  ▼
              IDLE                              HANDLING EVENTS
         (waiting for                     (fetch, push, sync,
          events)                          message events)
                    │
                    ▼
               TERMINATED
         (browser may terminate idle SW
          to conserve resources)
          — NOT the same as unregistered —
          — will be restarted on next event —
```

### Key Lifecycle Rules

```
1. A new SW cannot activate while any old SW is still controlling pages
   → New SW waits in INSTALLED state
   → Unless: skipWaiting() is called (takes over immediately)

2. An activated SW controls pages only if:
   → The page was loaded AFTER the SW activated
   → OR clients.claim() is called (takes control of existing pages)

3. A SW can be terminated by the browser while idle
   → It is NOT destroyed — it's suspended to save memory
   → The next event (fetch, push, etc.) restarts it automatically

4. SW updates are checked:
   → Every 24 hours (browser-enforced)
   → On every navigation to a scoped page
   → When you call registration.update()
```

---

## 3. Registration

```javascript
// main.js — register the Service Worker
async function registerServiceWorker() {
  if (!("serviceWorker" in navigator)) {
    console.log("Service Workers not supported");
    return;
  }

  try {
    const registration = await navigator.serviceWorker.register("/sw.js", {
      scope: "/", // which URLs this SW controls (default: same dir as sw.js)
      // updateViaCache: 'none' — bypass HTTP cache for SW script itself
      updateViaCache: "none",
    });

    console.log("SW registered. Scope:", registration.scope);

    // Check which state the SW is in
    if (registration.installing) {
      console.log("SW installing");
    } else if (registration.waiting) {
      console.log("SW waiting (new version available)");
      // User can be prompted to refresh for the new version
    } else if (registration.active) {
      console.log("SW active");
    }

    // Listen for future updates
    registration.addEventListener("updatefound", () => {
      const newWorker = registration.installing;
      newWorker.addEventListener("statechange", () => {
        if (
          newWorker.state === "installed" &&
          navigator.serviceWorker.controller
        ) {
          // New SW installed, old SW still controlling — prompt for refresh
          showUpdateAvailablePrompt();
        }
      });
    });
  } catch (err) {
    console.error("SW registration failed:", err);
  }
}

// Register after page load to not compete with initial resources
window.addEventListener("load", registerServiceWorker);
```

### Checking Current Controller

```javascript
// Is this page controlled by a Service Worker?
if (navigator.serviceWorker.controller) {
  console.log("Controlled by:", navigator.serviceWorker.controller.scriptURL);
} else {
  console.log("Not controlled (first load after registration)");
}

// Listen for controller changes (when SW updates and takes over)
navigator.serviceWorker.addEventListener("controllerchange", () => {
  console.log("New SW took control — might want to reload for latest version");
  window.location.reload(); // optional: force reload for fresh resources
});
```

---

## 4. Install Event — Pre-Caching

The `install` event fires once when the SW is first registered (or when a new version is detected). This is where you pre-cache critical resources.

```javascript
// sw.js
const CACHE_VERSION = "v2"; // bump this to trigger update
const CACHE_NAME = `app-${CACHE_VERSION}`;

// Critical resources to cache immediately on install
const PRECACHE_URLS = [
  "/", // HTML shell
  "/index.html",
  "/app.js", // main JS bundle
  "/styles.css", // main CSS
  "/logo.svg",
  "/offline.html", // fallback page for offline
  "/fonts/inter.woff2", // critical font
];

self.addEventListener("install", (event) => {
  console.log("[SW] Installing...");

  event.waitUntil(
    (async () => {
      const cache = await caches.open(CACHE_NAME);

      // Cache all critical assets
      // If ANY fail, the entire install fails (SW goes REDUNDANT)
      await cache.addAll(PRECACHE_URLS);

      console.log("[SW] Pre-cached", PRECACHE_URLS.length, "resources");

      // Force activation immediately (don't wait for old SW to unload)
      // Use carefully — see Section 12 for tradeoffs
      await self.skipWaiting();
    })(),
  );
});
```

### Resilient Pre-Caching

```javascript
// Cache what you can — don't fail install for optional resources
self.addEventListener("install", (event) => {
  event.waitUntil(
    (async () => {
      const cache = await caches.open(CACHE_NAME);

      // Required assets — install fails if these fail
      await cache.addAll(CRITICAL_URLS);

      // Optional assets — log failures but don't fail install
      const optionalResults = await Promise.allSettled(
        OPTIONAL_URLS.map((url) => cache.add(url)),
      );

      optionalResults.forEach((result, i) => {
        if (result.status === "rejected") {
          console.warn("[SW] Failed to cache optional:", OPTIONAL_URLS[i]);
        }
      });

      await self.skipWaiting();
    })(),
  );
});
```

---

## 5. Activate Event — Cache Cleanup

The `activate` event fires when the SW takes control. This is where you clean up old caches from previous versions.

```javascript
self.addEventListener("activate", (event) => {
  console.log("[SW] Activating...");

  event.waitUntil(
    (async () => {
      // Clean up old cache versions
      const cacheNames = await caches.keys();

      await Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME) // keep only current version
          .map((name) => {
            console.log("[SW] Deleting old cache:", name);
            return caches.delete(name);
          }),
      );

      // Take control of all open pages immediately
      // Without this, pages opened before activation aren't controlled
      await self.clients.claim();

      console.log(
        "[SW] Activated. Controlling",
        (await self.clients.matchAll()).length,
        "clients",
      );
    })(),
  );
});
```

### Cache Versioning Strategy

```javascript
// Naming convention:
const STATIC_CACHE = `static-${CACHE_VERSION}`; // HTML, JS, CSS, fonts
const DYNAMIC_CACHE = `dynamic-v1`; // API responses (versioned differently)
const IMAGE_CACHE = `images-v1`; // Images (long TTL)

// On activate: only delete caches that match your prefix, not all caches
const MY_CACHE_PREFIX = "myapp-";

const cacheNames = await caches.keys();
await Promise.all(
  cacheNames
    .filter((name) => name.startsWith(MY_CACHE_PREFIX) && name !== CACHE_NAME)
    .map((name) => caches.delete(name)),
);
```

---

## 6. Fetch Event — Network Interception

The `fetch` event fires for every request made by any page controlled by the SW — HTML navigation, API calls, images, fonts, scripts, everything.

```javascript
self.addEventListener("fetch", (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // Only handle GET requests — let others (POST, PUT, DELETE) pass through
  if (request.method !== "GET") return;

  // Only handle same-origin + configured external origins
  if (
    url.origin !== self.location.origin &&
    !ALLOWED_EXTERNAL_ORIGINS.includes(url.origin)
  ) {
    return; // don't intercept — pass through to network
  }

  event.respondWith(handleFetch(request));
});

async function handleFetch(request) {
  const url = new URL(request.url);

  // Route to appropriate caching strategy based on URL pattern
  if (isAPIRequest(url)) return networkFirst(request);
  if (isImageRequest(url))
    return cacheFirst(request, { cacheName: IMAGE_CACHE });
  if (isStaticAsset(url))
    return cacheFirst(request, { cacheName: STATIC_CACHE });
  if (isHTMLNavigation(request))
    return networkFirst(request, { fallback: "/offline.html" });

  return fetch(request); // default: pass through
}

function isAPIRequest(url) {
  return url.pathname.startsWith("/api/");
}

function isImageRequest(url) {
  return /\.(png|jpg|jpeg|gif|webp|svg|ico)$/i.test(url.pathname);
}

function isStaticAsset(url) {
  return /\.(js|css|woff2?|ttf|otf)$/i.test(url.pathname);
}

function isHTMLNavigation(request) {
  return request.mode === "navigate";
}
```

---

## 7. Caching Strategies

These are the five standard caching strategies. Most real apps use different strategies for different resource types.

### Strategy 1 — Cache First (Offline First)

```
Request → Check cache → Cache hit? Return immediately.
                      → Cache miss? Fetch network → Cache response → Return.

Best for: Static assets (JS, CSS, fonts, images)
          Resources with content-hash filenames (never stale)
Tradeoff: Serves stale content until cache is explicitly updated
```

```javascript
async function cacheFirst(request, { cacheName = STATIC_CACHE } = {}) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);

  if (cached) {
    return cached; // serve from cache immediately
  }

  try {
    const response = await fetch(request);

    if (response.ok) {
      // Cache for next time
      cache.put(request, response.clone());
    }

    return response;
  } catch (err) {
    // Network failed and nothing in cache
    throw new Error(`Cache first failed for ${request.url}: ${err.message}`);
  }
}
```

### Strategy 2 — Network First (Freshness First)

```
Request → Fetch network → Success? Cache response → Return.
                        → Failure? Check cache → Return cached or error.

Best for: API data, HTML pages, content that changes frequently
Tradeoff: Slower (waits for network), falls back to potentially stale cache
```

```javascript
async function networkFirst(
  request,
  { cacheName = DYNAMIC_CACHE, fallback } = {},
) {
  const cache = await caches.open(cacheName);

  try {
    const response = await fetch(request);

    if (response.ok) {
      cache.put(request, response.clone()); // update cache in background
    }

    return response;
  } catch (err) {
    // Network failed — try cache
    const cached = await cache.match(request);
    if (cached) return cached;

    // No cache — serve fallback if provided
    if (fallback) {
      const fallbackResponse = await caches.match(fallback);
      if (fallbackResponse) return fallbackResponse;
    }

    // Nothing — return a proper error response
    return new Response("Network error", {
      status: 503,
      statusText: "Service Unavailable",
    });
  }
}
```

### Strategy 3 — Stale While Revalidate

```
Request → Check cache → Cache hit? Return immediately (stale is fine)
                                   Fetch network in background → Update cache
                      → Cache miss? Fetch network → Cache → Return.

Best for: Non-critical content where freshness is nice but not required
          News feeds, social content, profile images
Tradeoff: User may see stale content; fresh version available on next request
```

```javascript
async function staleWhileRevalidate(
  request,
  { cacheName = DYNAMIC_CACHE } = {},
) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);

  // Revalidate in the background regardless
  const networkFetch = fetch(request)
    .then((response) => {
      if (response.ok) {
        cache.put(request, response.clone());
      }
      return response;
    })
    .catch(() => null); // network failure is OK — we have cache

  // Return cached immediately, or wait for network if no cache
  return cached ?? networkFetch;
}
```

### Strategy 4 — Cache Only

```
Request → Check cache → Return (or fail if not cached)
Never touches the network.

Best for: Offline-only resources, pre-cached assets during install
```

```javascript
async function cacheOnly(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  return new Response("Not in cache", { status: 504 });
}
```

### Strategy 5 — Network Only

```
Request → Fetch network → Return (bypass cache entirely)
No caching involved.

Best for: Analytics, POST requests, real-time data that must be fresh
```

```javascript
async function networkOnly(request) {
  return fetch(request); // bypass SW cache entirely
}
```

### Strategy Decision Tree

```
Is this a POST/PUT/DELETE?
  → Network Only (don't cache mutations)

Is this a content-hashed static asset (/app.abc123.js)?
  → Cache First (hash guarantees freshness)

Is this an API request for real-time data?
  → Network First (freshness matters, cache as fallback)

Is this a non-critical API (social feed, recommendations)?
  → Stale While Revalidate (fast + eventually fresh)

Is this an image or font that rarely changes?
  → Cache First with long TTL

Is this a navigation request (HTML)?
  → Network First with offline fallback
```

---

## 8. Cache API — Deep Dive

The Cache API is a key-value store for Request → Response pairs, available in Service Workers and regular pages.

```javascript
// Open a cache (creates it if it doesn't exist)
const cache = await caches.open("my-cache-v1");

// Add a single URL (fetches and stores)
await cache.add("/api/data");

// Add multiple URLs at once (atomic — all or nothing)
await cache.addAll(["/api/users", "/api/posts", "/api/config"]);

// Manually store a request-response pair
const request = new Request("/custom", { method: "GET" });
const response = new Response(JSON.stringify({ data: "value" }), {
  headers: { "Content-Type": "application/json" },
});
await cache.put(request, response);

// Retrieve a cached response
const match = await cache.match("/api/data");
// match is a Response object or undefined

// Match with options
const exactMatch = await cache.match("/api/data", {
  ignoreSearch: false, // don't ignore query params (default)
  ignoreMethod: false, // only match GET (default)
  ignoreVary: false, // respect Vary header (default)
});

// Delete a specific entry
const deleted = await cache.delete("/api/old-data");

// List all cached requests
const requests = await cache.keys();
requests.forEach((req) => console.log(req.url));

// Check if cache exists
const exists = await caches.has("my-cache-v1");

// List all cache names
const names = await caches.keys();

// Delete an entire cache
await caches.delete("my-cache-v1");
```

### Cache with TTL (Time-To-Live)

The Cache API doesn't natively support TTL. Implement it by storing metadata alongside the response:

```javascript
class TTLCache {
  constructor(cacheName, defaultTTLSeconds = 300) {
    this._cacheName = cacheName;
    this._defaultTTL = defaultTTLSeconds * 1000;
    this._metaCacheName = `${cacheName}-meta`;
  }

  async get(request) {
    const [cache, metaCache] = await Promise.all([
      caches.open(this._cacheName),
      caches.open(this._metaCacheName),
    ]);

    const metaResponse = await metaCache.match(request);
    if (metaResponse) {
      const meta = await metaResponse.json();
      if (Date.now() > meta.expiresAt) {
        // Expired — delete both cache entries
        await Promise.all([cache.delete(request), metaCache.delete(request)]);
        return null;
      }
    }

    return cache.match(request);
  }

  async set(request, response, ttlSeconds = this._defaultTTL / 1000) {
    const [cache, metaCache] = await Promise.all([
      caches.open(this._cacheName),
      caches.open(this._metaCacheName),
    ]);

    const meta = { expiresAt: Date.now() + ttlSeconds * 1000 };

    await Promise.all([
      cache.put(request, response.clone()),
      metaCache.put(
        request,
        new Response(JSON.stringify(meta), {
          headers: { "Content-Type": "application/json" },
        }),
      ),
    ]);
  }
}
```

---

## 9. Background Sync

Background Sync lets you defer network requests until the user has connectivity — even if they close the browser.

```javascript
// main.js — register a sync when network fails
async function sendAnalyticsEvent(event) {
  try {
    await fetch("/api/analytics", {
      method: "POST",
      body: JSON.stringify(event),
      headers: { "Content-Type": "application/json" },
    });
  } catch (err) {
    // Network failed — save to IndexedDB and register background sync
    await saveEventToIDB(event);

    const registration = await navigator.serviceWorker.ready;
    await registration.sync.register("sync-analytics");
    // 'sync-analytics' will fire in SW when connectivity returns
    // Even if the user closes the tab!
  }
}
```

```javascript
// sw.js — handle the sync event
self.addEventListener("sync", (event) => {
  if (event.tag === "sync-analytics") {
    event.waitUntil(syncAnalyticsEvents());
  }
});

async function syncAnalyticsEvents() {
  const pendingEvents = await getEventsFromIDB();

  const results = await Promise.allSettled(
    pendingEvents.map((event) =>
      fetch("/api/analytics", {
        method: "POST",
        body: JSON.stringify(event),
        headers: { "Content-Type": "application/json" },
      }),
    ),
  );

  // Remove successfully sent events
  const successfulIndices = results
    .map((r, i) => (r.status === "fulfilled" ? i : -1))
    .filter((i) => i !== -1);

  await removeEventsFromIDB(successfulIndices);

  // If any failed, throw to retry the sync
  const failures = results.filter((r) => r.status === "rejected");
  if (failures.length > 0) {
    throw new Error(`${failures.length} events failed to sync`);
  }
}
```

### Periodic Background Sync

```javascript
// main.js — register periodic sync (requires permission)
const registration = await navigator.serviceWorker.ready;

const status = await navigator.permissions.query({
  name: "periodic-background-sync",
});
if (status.state === "granted") {
  await registration.periodicSync.register("content-refresh", {
    minInterval: 24 * 60 * 60 * 1000, // at most once per day
  });
}
```

```javascript
// sw.js
self.addEventListener("periodicsync", (event) => {
  if (event.tag === "content-refresh") {
    event.waitUntil(refreshCachedContent());
  }
});
```

---

## 10. Push Notifications

Service Workers receive push messages even when the page is closed, enabling server-initiated notifications.

### Subscribing to Push

```javascript
// main.js
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready;

  // VAPID public key (from your push server)
  const VAPID_PUBLIC_KEY = "BExample...";

  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true, // required — push must show a notification
    applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
  });

  // Send subscription to your server
  await fetch("/api/push/subscribe", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(subscription),
  });

  return subscription;
}

function urlBase64ToUint8Array(base64String) {
  const padding = "=".repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, "+").replace(/_/g, "/");
  const rawData = atob(base64);
  return Uint8Array.from([...rawData].map((c) => c.charCodeAt(0)));
}
```

### Handling Push in Service Worker

```javascript
// sw.js
self.addEventListener("push", (event) => {
  if (!event.data) return;

  const data = event.data.json();

  const options = {
    body: data.body,
    icon: data.icon ?? "/icon-192.png",
    badge: "/badge-72.png",
    image: data.image,
    data: { url: data.url }, // passed to notificationclick
    actions: [
      { action: "open", title: "Open" },
      { action: "dismiss", title: "Dismiss" },
    ],
    tag: data.tag, // replace previous notification with same tag
    renotify: false, // don't vibrate again if tag matches
  };

  event.waitUntil(self.registration.showNotification(data.title, options));
});

// Handle notification clicks
self.addEventListener("notificationclick", (event) => {
  event.notification.close();

  if (event.action === "dismiss") return;

  const url = event.notification.data?.url ?? "/";

  event.waitUntil(
    self.clients.matchAll({ type: "window" }).then((clients) => {
      // If app is already open, focus the existing window
      const existing = clients.find((c) => c.url === url && "focus" in c);
      if (existing) return existing.focus();

      // Otherwise open a new window
      return self.clients.openWindow(url);
    }),
  );
});
```

---

## 11. Service Worker Communication

### Page → Service Worker

```javascript
// main.js — send message to active Service Worker
navigator.serviceWorker.controller?.postMessage({
  type: "SKIP_WAITING", // tell SW to activate immediately
});

// Or via the registration:
const registration = await navigator.serviceWorker.ready;
registration.active?.postMessage({ type: "CLEAR_CACHE" });
```

### Service Worker → Page (Broadcast)

```javascript
// sw.js — send message to all controlled pages
async function broadcastToClients(message) {
  const clients = await self.clients.matchAll({
    type: "window",
    includeUncontrolled: false,
  });

  clients.forEach((client) => client.postMessage(message));
}

// Example: notify pages when cache is updated
self.addEventListener("fetch", (event) => {
  event.respondWith(
    staleWhileRevalidate(event.request).then(async (response) => {
      if (response.url && isFresh(response)) {
        await broadcastToClients({
          type: "CACHE_UPDATED",
          url: response.url,
        });
      }
      return response;
    }),
  );
});
```

```javascript
// main.js — listen for messages from SW
navigator.serviceWorker.addEventListener("message", (event) => {
  if (event.data.type === "CACHE_UPDATED") {
    console.log("Content updated:", event.data.url);
    // Optionally notify the user or reload
  }
});
```

### MessageChannel for Two-Way Communication

```javascript
// main.js — two-way with a MessageChannel
function sendAndReceive(message) {
  return new Promise((resolve, reject) => {
    const { port1, port2 } = new MessageChannel();

    port1.onmessage = ({ data }) => {
      if (data.error) reject(new Error(data.error));
      else resolve(data.result);
    };

    navigator.serviceWorker.controller?.postMessage(message, [port2]);
  });
}

// Usage
const cacheStats = await sendAndReceive({ type: "GET_CACHE_STATS" });
```

```javascript
// sw.js — receive and respond via the MessageChannel port
self.addEventListener("message", (event) => {
  if (event.data.type === "GET_CACHE_STATS") {
    const port = event.ports[0];

    caches
      .keys()
      .then(async (names) => {
        const stats = await Promise.all(
          names.map(async (name) => {
            const cache = await caches.open(name);
            const keys = await cache.keys();
            return { name, count: keys.length };
          }),
        );
        port.postMessage({ result: stats });
      })
      .catch((err) => {
        port.postMessage({ error: err.message });
      });
  }
});
```

---

## 12. Updating a Service Worker

### The Update Problem

```
User has v1 SW active, serving your app.
You deploy v2 of your app.
The browser detects the new sw.js → starts installing v2.
BUT: v1 is still controlling all open tabs.
     v2 waits in INSTALLED state until all v1 tabs are closed.

Problem: Users with open tabs may not see v2 for days.
```

### Strategy 1 — `skipWaiting` (Aggressive)

```javascript
// sw.js — take control immediately on install
self.addEventListener("install", (event) => {
  event.waitUntil(
    caches
      .open(CACHE_NAME)
      .then((cache) => cache.addAll(PRECACHE_URLS))
      .then(() => self.skipWaiting()), // don't wait for old SW
  );
});

self.addEventListener("activate", (event) => {
  event.waitUntil(self.clients.claim()); // take control of all pages
});
```

**Risk:** If the new SW serves new resources but the page has already loaded old resources, there can be version mismatch. Safe for single-page apps with full resource hashes.

### Strategy 2 — Prompt User to Refresh

```javascript
// main.js — detect waiting SW, prompt user
let waitingSW = null;

navigator.serviceWorker.addEventListener("controllerchange", () => {
  window.location.reload(); // reload when new SW takes over
});

const registration = await navigator.serviceWorker.register("/sw.js");

registration.addEventListener("updatefound", () => {
  const newSW = registration.installing;

  newSW.addEventListener("statechange", () => {
    if (newSW.state === "installed" && navigator.serviceWorker.controller) {
      waitingSW = newSW;
      showUpdateBanner(); // "New version available! Click to update."
    }
  });
});

// When user clicks "Update":
function applyUpdate() {
  waitingSW?.postMessage({ type: "SKIP_WAITING" });
}
```

```javascript
// sw.js — listen for SKIP_WAITING message
self.addEventListener("message", (event) => {
  if (event.data.type === "SKIP_WAITING") {
    self.skipWaiting(); // now activate
  }
});
```

---

## 13. Service Worker Scope

The scope defines which URLs the SW controls. Default scope is the directory of the SW file.

```javascript
// sw.js at /app/sw.js → default scope is /app/
// Controls: /app/, /app/index.html, /app/products/...
// Does NOT control: /, /blog/, /admin/

// Override scope during registration:
navigator.serviceWorker.register("/sw.js", {
  scope: "/app/", // explicitly set scope
});

// Scope can be narrower than SW file location
// but NEVER broader (requires Service-Worker-Allowed header)
```

### Scope Header for Broader Scope

```
# To allow a SW at /app/sw.js to control /
# The server must send this header when serving sw.js:
Service-Worker-Allowed: /
```

---

## 14. Debugging Service Workers

### Chrome DevTools

```
Application tab → Service Workers:
  - See all registered SWs for the current origin
  - Status: activated and running / waiting / stopped
  - "Update on reload" — forces SW update on every page reload (dev mode)
  - "Bypass for network" — skip SW entirely (dev mode)
  - "Offline" checkbox — simulate offline
  - Inspect: opens Sources debugger in SW context
  - Unregister: remove the SW
  - Push: simulate a push notification
  - Sync: trigger background sync manually

Application tab → Cache Storage:
  - Browse all cached requests/responses
  - Delete individual entries or entire caches

Application tab → Storage → Clear site data:
  - Wipe all caches, cookies, IDB, localStorage
```

### Programmatic Debugging

```javascript
// sw.js — verbose logging in development
const DEBUG = self.location.hostname === "localhost";

function log(...args) {
  if (DEBUG) console.log("[SW]", ...args);
}

self.addEventListener("fetch", (event) => {
  log("Fetch:", event.request.method, event.request.url);
  event.respondWith(
    handleFetch(event.request).then((response) => {
      log(
        "Response:",
        response.status,
        event.request.url,
        response.headers.get("X-Cache-Status") ?? "network",
      );
      return response;
    }),
  );
});
```

### Checking SW Status from Page

```javascript
async function getServiceWorkerStatus() {
  if (!("serviceWorker" in navigator)) return "not-supported";

  const registration = await navigator.serviceWorker.getRegistration();
  if (!registration) return "not-registered";

  if (registration.installing) return "installing";
  if (registration.waiting) return "waiting";
  if (registration.active) return "active";

  return "unknown";
}
```

---

## 15. Production Architecture

### Complete Service Worker for a SPA

```javascript
// sw.js — production-ready SPA service worker
const APP_VERSION = "2.1.0";
const STATIC_CACHE = `static-${APP_VERSION}`;
const API_CACHE = "api-cache-v1";
const IMAGE_CACHE = "image-cache-v1";

const STATIC_URLS = [
  "/",
  "/index.html",
  "/app.js",
  "/styles.css",
  "/offline.html",
];

// ─── Install ─────────────────────────────────────────────────────
self.addEventListener("install", (event) => {
  event.waitUntil(
    caches
      .open(STATIC_CACHE)
      .then((cache) => cache.addAll(STATIC_URLS))
      .then(() => self.skipWaiting()),
  );
});

// ─── Activate ────────────────────────────────────────────────────
self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches
      .keys()
      .then((names) =>
        Promise.all(
          names
            .filter((n) => n.startsWith("static-") && n !== STATIC_CACHE)
            .map((n) => caches.delete(n)),
        ),
      )
      .then(() => self.clients.claim()),
  );
});

// ─── Fetch ───────────────────────────────────────────────────────
self.addEventListener("fetch", (event) => {
  const url = new URL(event.request.url);

  // Skip non-GET
  if (event.request.method !== "GET") return;

  // Skip DevTools / browser internals
  if (url.protocol !== "https:" && url.hostname !== "localhost") return;

  if (url.pathname.startsWith("/api/")) {
    event.respondWith(networkFirst(event.request, API_CACHE));
  } else if (/\.(png|jpg|jpeg|gif|webp|avif|svg)$/.test(url.pathname)) {
    event.respondWith(
      cacheFirst(event.request, IMAGE_CACHE, { ttl: 7 * 24 * 60 * 60 }),
    );
  } else if (event.request.mode === "navigate") {
    event.respondWith(
      networkFirst(event.request, STATIC_CACHE).catch(() =>
        caches.match("/offline.html"),
      ),
    );
  } else {
    event.respondWith(cacheFirst(event.request, STATIC_CACHE));
  }
});

// ─── Message ─────────────────────────────────────────────────────
self.addEventListener("message", (event) => {
  if (event.data.type === "SKIP_WAITING") self.skipWaiting();
  if (event.data.type === "CLEAR_API_CACHE") caches.delete(API_CACHE);
});

// ─── Strategy Implementations ────────────────────────────────────
async function cacheFirst(request, cacheName) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  if (response.ok) cache.put(request, response.clone());
  return response;
}

async function networkFirst(request, cacheName) {
  const cache = await caches.open(cacheName);
  try {
    const response = await fetch(request);
    if (response.ok) cache.put(request, response.clone());
    return response;
  } catch {
    const cached = await cache.match(request);
    if (cached) return cached;
    return new Response("Offline", { status: 503 });
  }
}
```

---

## 16. Good Practices

### ✅ Version your caches with your app version

```javascript
// ✅ Every deploy bumps CACHE_VERSION — old caches cleaned on activate
const CACHE_VERSION = "v2.1.0";
const CACHE_NAME = `app-${CACHE_VERSION}`;
```

### ✅ Always provide an offline fallback for navigations

```javascript
// ✅ Users who navigate offline see a real page, not a browser error
if (event.request.mode === "navigate") {
  event.respondWith(
    fetch(event.request).catch(() => caches.match("/offline.html")),
  );
}
```

### ✅ Use `updateViaCache: 'none'` for SW script

```javascript
// ✅ SW script bypasses HTTP cache — ensures updates are detected promptly
navigator.serviceWorker.register("/sw.js", { updateViaCache: "none" });
```

### ✅ Keep SW script slim — put logic in separate modules

```javascript
// ✅ Worker imports for modular SW code
importScripts("/sw-strategies.js"); // caching strategies
importScripts("/sw-push.js"); // push notification handling
importScripts("/sw-sync.js"); // background sync
```

### ✅ Set appropriate cache size limits

```javascript
// ✅ Trim old entries when cache gets too large
async function trimCache(cacheName, maxItems) {
  const cache = await caches.open(cacheName);
  const requests = await cache.keys();

  if (requests.length > maxItems) {
    // Delete oldest entries (LRU: keys are in insertion order)
    await Promise.all(
      requests.slice(0, requests.length - maxItems).map((r) => cache.delete(r)),
    );
  }
}
```

---

## 17. Bad Practices

### ❌ Caching POST/PUT/DELETE responses

```javascript
// ❌ Never cache mutation requests
self.addEventListener("fetch", (event) => {
  // Intercept ALL requests including POST
  event.respondWith(cacheFirst(event.request)); // caching POST = wrong!
});

// ✅ Only cache GET requests
if (event.request.method !== "GET") return; // skip non-GET
```

### ❌ Calling `skipWaiting()` unconditionally without coordination

```javascript
// ❌ New SW activates immediately — but page loaded with old assets
// If JS/CSS files changed, page now has old HTML + new assets = broken
self.addEventListener("install", (event) => {
  event.waitUntil(caches.open(CACHE_NAME).then((c) => c.addAll(URLS)));
  self.skipWaiting(); // outside waitUntil — races with caching
});

// ✅ Inside waitUntil, coordinate with page
event.waitUntil(
  caches
    .open(CACHE_NAME)
    .then((c) => c.addAll(URLS))
    .then(() => self.skipWaiting()), // only after caching is complete
);
```

### ❌ Caching everything with the same strategy

```javascript
// ❌ Network-first for static assets wastes bandwidth
// ❌ Cache-first for API data serves stale data
self.addEventListener("fetch", (event) => {
  event.respondWith(networkFirst(event.request)); // same strategy for all
});
```

### ❌ Not handling SW errors in registration

```javascript
// ❌ Registration failure silently ignored
navigator.serviceWorker.register("/sw.js");

// ✅ Handle failure gracefully
navigator.serviceWorker
  .register("/sw.js")
  .catch((err) => console.error("SW failed to register:", err));
```

---

## 18. Common Mistakes

### Mistake 1 — Expecting SW to work on first load

```javascript
// On the very first page load after registration:
// The SW installs and activates, but it doesn't control the current page
// (unless clients.claim() is called in activate)
// The SW only controls pages LOADED AFTER it activates

// Fix: call clients.claim() in activate event
self.addEventListener("activate", (event) => {
  event.waitUntil(self.clients.claim()); // control existing pages too
});
```

### Mistake 2 — SW scope too narrow

```javascript
// sw.js at /app/sw.js
// This SW does NOT control /
// It only controls /app/ and below

// If your app routes include /, registration at /app/sw.js won't intercept root
// Fix: put sw.js at /sw.js, or use Service-Worker-Allowed header
```

### Mistake 3 — Not cloning responses before caching

```javascript
// ❌ Response body can only be read ONCE
event.respondWith(
  fetch(event.request).then(async (response) => {
    const cache = await caches.open(CACHE_NAME);
    cache.put(event.request, response); // puts the response in cache
    return response; // ❌ body already consumed by cache.put!
  }),
);

// ✅ Clone before caching — original returned to page, clone goes to cache
event.respondWith(
  fetch(event.request).then(async (response) => {
    const cache = await caches.open(CACHE_NAME);
    cache.put(event.request, response.clone()); // cache the clone
    return response; // ✅ original returned to page
  }),
);
```

### Mistake 4 — Registering SW inside a try/catch that swallows errors

```javascript
// ❌ Errors silently swallowed
try {
  navigator.serviceWorker.register("/sw.js");
} catch (e) {} // error lost
```

---

## 19. Interview-Level Explanation

> **"What is a Service Worker? How does it differ from other workers? What can you build with it?"**

**Strong answer:**

> "A Service Worker is a JavaScript file that runs in the background as a network proxy between a web page and the internet. Unlike Dedicated or Shared Workers, a Service Worker survives after the page closes — it can receive push notifications, handle background sync, and cache resources even when the user isn't actively using the app. It only works on HTTPS for security reasons.
>
> The lifecycle is explicit: the browser downloads the SW script, fires an 'install' event where you pre-cache critical resources, then fires 'activate' where you clean up old caches. The SW doesn't control existing pages until they're reloaded, unless you call `clients.claim()` in the activate handler.
>
> The most powerful feature is the 'fetch' event — it fires for every network request from controlled pages, letting you return responses from cache, the network, or generate them dynamically. This is where caching strategies live. Cache First serves static assets instantly from cache. Network First tries the network first and falls back to cache — good for API data. Stale While Revalidate returns the cached version immediately and updates in the background — good for content where freshness is nice but not critical.
>
> The classic mistake is not cloning responses before caching — a Response body can only be consumed once. You call `response.clone()` to get a copy: one goes to the cache, one goes back to the page.
>
> For updates, the browser installs the new SW while the old one is still running. The new SW waits in 'installed' state until all tabs are closed. You can force immediate activation with `skipWaiting()` and `clients.claim()`, or prompt the user to refresh. For production, the safer approach is prompting the user with a 'New version available' banner."

---

## 20. Exercises

### Exercise 1 — Implement a complete fetch handler

Write a fetch event handler that:

- Applies Network First to all `/api/` requests with a 5-second timeout
- Applies Cache First to all `.js`, `.css`, and font files
- Applies Stale While Revalidate to all image requests
- Returns `/offline.html` for navigation requests when offline

<details>
<summary>Solution</summary>

```javascript
self.addEventListener("fetch", (event) => {
  if (event.request.method !== "GET") return;

  const url = new URL(event.request.url);

  if (url.pathname.startsWith("/api/")) {
    event.respondWith(networkFirstWithTimeout(event.request, 5000));
    return;
  }

  if (/\.(js|css|woff2?)$/.test(url.pathname)) {
    event.respondWith(cacheFirst(event.request));
    return;
  }

  if (/\.(png|jpg|jpeg|gif|webp|svg)$/.test(url.pathname)) {
    event.respondWith(staleWhileRevalidate(event.request));
    return;
  }

  if (event.request.mode === "navigate") {
    event.respondWith(
      fetch(event.request).catch(() => caches.match("/offline.html")),
    );
    return;
  }
});

async function networkFirstWithTimeout(request, timeoutMs) {
  const cache = await caches.open("api-cache");

  const networkPromise = fetch(request).then(async (response) => {
    if (response.ok) cache.put(request, response.clone());
    return response;
  });

  const timeoutPromise = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("timeout")), timeoutMs),
  );

  try {
    return await Promise.race([networkPromise, timeoutPromise]);
  } catch {
    return (
      (await cache.match(request)) ??
      new Response("Unavailable", { status: 503 })
    );
  }
}
```

</details>

---

### Exercise 2 — Offline form submission with Background Sync

Build a feedback form that:

- Submits immediately when online
- Saves to IndexedDB when offline
- Automatically retries when connectivity returns

<details>
<summary>Solution sketch</summary>

```javascript
// main.js
async function submitFeedback(formData) {
  try {
    await fetch("/api/feedback", {
      method: "POST",
      body: JSON.stringify(formData),
      headers: { "Content-Type": "application/json" },
    });
    showSuccess("Feedback submitted!");
  } catch {
    // Save to IDB
    await savePendingFeedback(formData);

    // Register background sync
    const reg = await navigator.serviceWorker.ready;
    await reg.sync.register("sync-feedback");

    showSuccess("Saved! Will submit when online.");
  }
}

// sw.js
self.addEventListener("sync", (event) => {
  if (event.tag === "sync-feedback") {
    event.waitUntil(syncPendingFeedback());
  }
});

async function syncPendingFeedback() {
  const pending = await getPendingFeedback(); // from IDB

  for (const item of pending) {
    const res = await fetch("/api/feedback", {
      method: "POST",
      body: JSON.stringify(item.data),
      headers: { "Content-Type": "application/json" },
    });

    if (res.ok) await removePendingFeedback(item.id);
  }
}
```

</details>

---

## 🔗 Related Topics

- [`javascript-core/12-web-workers.md`](./12-web-workers.md) — Dedicated and Shared Workers
- [`caching/02-service-worker-cache.md`](../caching/02-service-worker-cache.md) — Advanced caching patterns
- [`networking/04-prefetching-preloading.md`](../networking/04-prefetching-preloading.md) — Prefetching with SW
- [`browser-internals/09-browser-caching.md`](../browser-internals/09-browser-caching.md) — HTTP caching vs SW caching

---

<div align="center">

**Next:** [`javascript-core/14-observer-patterns.md`](./14-observer-patterns.md) →

</div>
