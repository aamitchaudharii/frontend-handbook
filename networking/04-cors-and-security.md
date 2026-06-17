# 04 — CORS and Network Security

> **"CORS exists because the web's open nature — any page can load any resource from anywhere — becomes a security liability the moment you add authenticated APIs. CORS is the browser's mechanism for enforcing server consent: I will only let JavaScript on page A read a response from server B if server B explicitly says that's okay."**

CORS (Cross-Origin Resource Sharing) is the security mechanism that governs cross-origin requests in browsers. It is simultaneously one of the most misunderstood topics in web development (developers often fight it instead of understanding it) and one of the most security-critical (misconfiguring it can expose user data to any malicious website). This document covers how CORS works at the protocol level, common misconfigurations, other network security mechanisms (CSP, SRI, certificate pinning, mixed content), and how to configure all of them correctly.

---

## 📚 Table of Contents

1. [The Same-Origin Policy](#1-the-same-origin-policy)
2. [What CORS Actually Does](#2-what-cors-actually-does)
3. [Simple vs Preflighted Requests](#3-simple-vs-preflighted-requests)
4. [CORS Headers Reference](#4-cors-headers-reference)
5. [CORS and Credentials](#5-cors-and-credentials)
6. [Common CORS Configurations](#6-common-cors-configurations)
7. [CORS Misconfigurations and Vulnerabilities](#7-cors-misconfigurations-and-vulnerabilities)
8. [Content Security Policy (CSP)](#8-content-security-policy-csp)
9. [Subresource Integrity (SRI)](#9-subresource-integrity-sri)
10. [Mixed Content](#10-mixed-content)
11. [Certificate Transparency and HSTS](#11-certificate-transparency-and-hsts)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Same-Origin Policy

The Same-Origin Policy (SOP) is the bedrock of browser security. It prevents a script from one origin from accessing resources from another origin.

### What an Origin Is

```
ORIGIN = scheme + host + port

https://app.example.com:443/path?query#hash
         ↑                  ↑   ↑
         host               port scheme is https

SAME ORIGIN comparisons:
  https://example.com    and  https://example.com         → SAME
  https://example.com    and  http://example.com          → DIFFERENT (scheme)
  https://example.com    and  https://www.example.com     → DIFFERENT (host)
  https://example.com    and  https://api.example.com     → DIFFERENT (subdomain)
  https://example.com    and  https://example.com:8080    → DIFFERENT (port)
  https://example.com:443 and https://example.com         → SAME (443 is implicit)
```

### What SOP Allows vs Blocks

```
ALLOWED BY DEFAULT (no CORS needed):
  ✓ Navigate to any URL (location.href = 'https://other.com')
  ✓ Load images (<img src="https://other.com/photo.jpg">)
  ✓ Load CSS (<link rel="stylesheet" href="https://cdn.com/styles.css">)
  ✓ Load scripts (<script src="https://cdn.com/lib.js"></script>)
  ✓ Embed iframes (<iframe src="https://other.com/page">)
  ✓ Form submissions (POST to any origin)

BLOCKED BY DEFAULT (requires CORS):
  ✗ XMLHttpRequest/fetch to different origin: reading the response
  ✗ Access to cross-origin iframe's DOM/localStorage
  ✗ Reading cross-origin canvas after tainted (drawImage from different origin)
```

### Why SOP Exists

```
WITHOUT SOP:
  Evil website loads:
  <script>
    fetch('https://your-bank.com/api/account')
      .then(r => r.json())
      .then(data => sendToEvil(data)); // steal your bank data!
  </script>

  The browser would send your bank cookies (credentials)
  → Bank API responds with YOUR account data
  → Evil site reads it
  → Account compromised

WITH SOP:
  fetch('https://your-bank.com/api/account') → browser makes request
  But: the browser REFUSES to give the response to the evil page's JavaScript
  Unless: your bank explicitly allows evil.com via CORS (it wouldn't)
```

---

## 2. What CORS Actually Does

CORS is NOT a security feature for servers — it's a browser-enforced policy that **lets servers relax** the Same-Origin Policy for specific origins.

```
IMPORTANT DISTINCTION:
  CORS is enforced by BROWSERS, not servers.
  Server-to-server requests: no CORS (not a browser)
  curl, Postman, backend services: bypass CORS entirely

CORS flow:
  Browser JavaScript on https://app.example.com
    → fetch('https://api.other-domain.com/data')
  Browser sends request (server receives and processes it)
  Server responds with (or without) CORS headers
  Browser checks: does the response include Access-Control-Allow-Origin?
    YES, includes app.example.com → give response to JavaScript
    NO → block response from JavaScript (request WAS sent, response is hidden)

CRITICAL: The server still PROCESSES the request either way
CORS does not prevent the server-side operation
CORS only controls what the BROWSER exposes to the script
```

---

## 3. Simple vs Preflighted Requests

### Simple Requests (No Preflight)

```
Conditions for a "simple" request (no OPTIONS preflight):
  Method: GET, HEAD, or POST
  Content-Type (if POST): one of:
    - application/x-www-form-urlencoded
    - multipart/form-data
    - text/plain
  Headers: only "CORS-safelisted" headers:
    - Accept, Accept-Language, Content-Language
    - Content-Type (limited values above)

Simple request flow:
  Browser → Server: GET /api/data (with Origin: https://app.example.com)
  Server → Browser: response + CORS headers (or not)
  Browser: check CORS headers → allow or block

  No preflight: request is ALWAYS sent
```

### Preflighted Requests

```
Conditions that trigger preflight:
  Method: PUT, DELETE, PATCH, or other non-simple methods
  Content-Type: application/json, application/xml, etc.
  ANY custom request header (Authorization, X-API-Key, X-Custom-...)

Preflight flow:
  Browser → Server: OPTIONS /api/data
    Origin: https://app.example.com
    Access-Control-Request-Method: PUT
    Access-Control-Request-Headers: Content-Type, Authorization

  Server → Browser: 200 OK (or 204)
    Access-Control-Allow-Origin: https://app.example.com
    Access-Control-Allow-Methods: PUT, POST, DELETE
    Access-Control-Allow-Headers: Content-Type, Authorization
    Access-Control-Max-Age: 86400

  If preflight succeeds:
  Browser → Server: PUT /api/data
    Origin: https://app.example.com
    Content-Type: application/json
    Authorization: Bearer ...

  Server → Browser: response + CORS headers
```

### Preflight Cost and Caching

```javascript
// Preflight adds a full round trip before the real request
// On a 100ms RTT connection:
//   With preflight: 100ms (OPTIONS) + 100ms (actual) = 200ms
//   Without preflight: 100ms (actual) = 100ms

// Caching preflights: Access-Control-Max-Age
// Server responds with max-age in seconds
Access-Control-Max-Age: 86400  // cache preflight for 24 hours

// Browser caches per (origin, method, headers) combination
// Within 24 hours: no OPTIONS request needed for same request shape

// Browser limits on preflight cache duration:
// Chrome: max 7200 seconds (2 hours)
// Firefox: max 86400 seconds (24 hours)
// Safari: max 600 seconds (10 minutes)
```

---

## 4. CORS Headers Reference

### Response Headers (Server → Browser)

```http
# Required: which origins are allowed to read the response
Access-Control-Allow-Origin: https://app.example.com
# OR (not recommended):
Access-Control-Allow-Origin: *   # any origin; but: cannot be used with credentials

# Required for preflight: which methods are allowed
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH, OPTIONS

# Required for preflight if client sends custom headers
Access-Control-Allow-Headers: Content-Type, Authorization, X-API-Key, X-Request-ID

# Preflight cache duration (seconds)
Access-Control-Max-Age: 86400

# Required for credentials (cookies, auth): must be true
Access-Control-Allow-Credentials: true
# Note: CANNOT combine with Access-Control-Allow-Origin: *

# Expose specific response headers to JavaScript
# By default: only basic headers are readable (Cache-Control, Content-Language,
# Content-Length, Content-Type, Expires, Last-Modified, Pragma)
Access-Control-Expose-Headers: X-RateLimit-Remaining, X-RateLimit-Reset, ETag
```

### Request Headers (Browser → Server, automatic)

```http
# Browser adds automatically to cross-origin requests
Origin: https://app.example.com         # always added to cross-origin requests

# Preflight only:
Access-Control-Request-Method: PUT      # method the actual request will use
Access-Control-Request-Headers: Authorization, Content-Type  # headers the actual request will include
```

---

## 5. CORS and Credentials

### Sending Credentials Cross-Origin

```javascript
// By default: fetch does NOT send cookies or auth headers cross-origin
fetch("https://api.other.com/user", {
  credentials: "same-origin", // default: only for same-origin
});

// Include credentials cross-origin:
fetch("https://api.other.com/user", {
  credentials: "include", // sends cookies AND Authorization header
});
// Server MUST respond with:
// Access-Control-Allow-Credentials: true
// Access-Control-Allow-Origin: https://app.example.com (NOT *)
```

### Server Configuration for Credentialed Requests

```javascript
// Express.js: CORS with credentials
const cors = require("cors");

app.use(
  cors({
    origin: (origin, callback) => {
      const allowedOrigins = [
        "https://app.example.com",
        "https://staging.example.com",
      ];

      if (!origin) return callback(null, true); // same-origin or non-browser
      if (allowedOrigins.includes(origin)) {
        return callback(null, origin); // dynamic: reflect the specific origin
      }
      callback(new Error(`CORS: origin ${origin} not allowed`));
    },
    credentials: true, // Access-Control-Allow-Credentials: true
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH"],
    allowedHeaders: ["Content-Type", "Authorization", "X-API-Key"],
    exposedHeaders: ["X-RateLimit-Remaining", "X-Total-Count"],
    maxAge: 86400, // Access-Control-Max-Age
  }),
);
```

---

## 6. Common CORS Configurations

### Public API (No Credentials Needed)

```http
# Public API: anyone can read responses, no cookies needed
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, HEAD
Access-Control-Max-Age: 86400
# Note: * means no credentials — cannot be used with credentials: include
```

### Same-Company Multiple Domains

```javascript
// Internal API serving multiple company frontends
const ALLOWED_ORIGINS = new Set([
  "https://app.example.com",
  "https://admin.example.com",
  "https://staging.example.com",
  "https://dev.example.com",
  // Localhost for development:
  "http://localhost:3000",
  "http://localhost:5173",
]);

function setCORSHeaders(req, res) {
  const origin = req.headers.origin;

  if (origin && ALLOWED_ORIGINS.has(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin); // reflect exact origin
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Vary", "Origin"); // ← CRITICAL: tell caches
  }

  if (req.method === "OPTIONS") {
    res.setHeader(
      "Access-Control-Allow-Methods",
      "GET, POST, PUT, DELETE, PATCH",
    );
    res.setHeader(
      "Access-Control-Allow-Headers",
      "Content-Type, Authorization",
    );
    res.setHeader("Access-Control-Max-Age", "86400");
    return res.status(204).end();
  }
}
```

### The Vary: Origin Header

```http
# CRITICAL when reflecting dynamic origins:
Access-Control-Allow-Origin: https://app.example.com  (dynamic)
Vary: Origin

# Without Vary: Origin:
# CDN may cache the response with Access-Control-Allow-Origin: https://app.example.com
# Then serve it to a request from https://admin.example.com
# admin.example.com: CORS error because its origin doesn't match the cached header

# With Vary: Origin:
# CDN creates separate cache entry per Origin value
# Each origin gets its own correct CORS header
```

---

## 7. CORS Misconfigurations and Vulnerabilities

### Vulnerability 1 — Reflecting Any Origin

```javascript
// ❌ CRITICAL VULNERABILITY: reflecting any origin unconditionally
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", req.headers.origin); // any origin!
  res.header("Access-Control-Allow-Credentials", "true");
  next();
});

// Attack:
// Attacker's site: https://evil.com
// fetch('https://api.example.com/user/profile', { credentials: 'include' })
// → Server reflects evil.com as allowed origin
// → Browser accepts response → attacker reads YOUR profile data

// ✅ Whitelist-based origin validation
if (ALLOWED_ORIGINS.has(req.headers.origin)) {
  res.header("Access-Control-Allow-Origin", req.headers.origin);
}
```

### Vulnerability 2 — Subdomain Wildcard Matching

```javascript
// ❌ Vulnerable: allows any subdomain
if (origin.endsWith('.example.com')) {
  res.header('Access-Control-Allow-Origin', origin);
}
// Attack: attacker registers evil.example.com → matches the pattern!

// ❌ Vulnerable: regex not anchored
if (/example\.com/.test(origin)) {
  res.header('Access-Control-Allow-Origin', origin);
}
// Attack: evil-example.com matches! (contains "example.com" as substring)

// ✅ Strict whitelist
const ALLOWED = new Set(['https://app.example.com', 'https://admin.example.com']);
if (ALLOWED.has(origin)) { ... }
```

### Vulnerability 3 — Null Origin

```javascript
// ❌ Allowing null origin
if (origin === "null" || ALLOWED.has(origin)) {
  res.header("Access-Control-Allow-Origin", origin);
}
// "null" origin is sent by:
// - Sandbox iframes: <iframe sandbox="...">
// - Local file: file:///evil.html
// Attacker can use: <iframe sandbox="allow-scripts allow-same-origin" src="data:...">

// ✅ Never allow null origin (unless specifically required for file:// apps)
```

### Vulnerability 4 — Overly Permissive for Sensitive Endpoints

```javascript
// ❌ Broad CORS on all routes, including sensitive ones
app.use(cors({ origin: "*" }));

// /api/admin, /api/user/export, /api/payment — all accessible from anywhere!

// ✅ CORS per-route: restrictive for sensitive, open for public
app.use("/api/public/", cors({ origin: "*" }));
app.use("/api/user/", cors({ origin: ALLOWED_ORIGINS }));
app.use("/api/admin/", cors({ origin: INTERNAL_ORIGINS }));
```

---

## 8. Content Security Policy (CSP)

CSP is a browser security feature that restricts which resources can be loaded. It mitigates XSS attacks by preventing injected scripts from running.

### CSP Header

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com 'nonce-abc123';
  style-src  'self' 'unsafe-inline' https://fonts.googleapis.com;
  img-src    'self' data: https://cdn.example.com;
  font-src   'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com wss://ws.example.com;
  frame-src  'none';
  object-src 'none';
  base-uri   'self';
  form-action 'self';
  upgrade-insecure-requests;
```

### CSP Directives

```
default-src:  Fallback for all resource types not explicitly listed
script-src:   JavaScript sources
style-src:    CSS sources
img-src:      Image sources
font-src:     Font sources
connect-src:  Fetch, XHR, WebSocket, EventSource destinations
frame-src:    iframe sources
object-src:   Plugin sources (<object>, <embed>)
base-uri:     Restricts <base href>
form-action:  Where forms can submit to
worker-src:   Web Worker sources

SPECIAL VALUES:
'self':         same origin as the page
'none':         no sources allowed
'unsafe-inline': inline scripts/styles (avoid — defeats XSS protection)
'unsafe-eval':  eval(), new Function() (avoid)
'nonce-{value}': specific inline script with matching nonce attribute
'sha256-{hash}': specific inline script matching the hash
https:           any HTTPS source
data:            data: URLs
wss:             WebSocket Secure sources
```

### Nonce-Based CSP

```javascript
// Server generates a fresh nonce per request
function generateNonce() {
  return crypto.randomBytes(16).toString("base64");
}

app.use((req, res, next) => {
  const nonce = generateNonce();
  res.locals.nonce = nonce;

  res.setHeader(
    "Content-Security-Policy",
    [
      `default-src 'self'`,
      `script-src 'self' 'nonce-${nonce}'`,
      `style-src 'self' 'nonce-${nonce}'`,
    ].join("; "),
  );

  next();
});

// In HTML template (nonce injected server-side):
res.send(`
  <html>
    <head>
      <script nonce="${nonce}" src="/app.js"></script>
      <style nonce="${nonce}">body { margin: 0; }</style>
    </head>
  </html>
`);
// Inline scripts without the correct nonce: blocked by CSP
```

### CSP Violation Reporting

```http
Content-Security-Policy:
  default-src 'self';
  report-uri /api/csp-report;

# Modern: report-to
Content-Security-Policy:
  default-src 'self';
  report-to csp-violations;

Report-To: {
  "group": "csp-violations",
  "max_age": 86400,
  "endpoints": [{"url": "https://report-collector.example.com/csp"}]
}
```

```javascript
// Server: collect and analyze CSP violations
app.post(
  "/api/csp-report",
  express.json({ type: "application/csp-report" }),
  (req, res) => {
    const report = req.body["csp-report"];
    console.warn("CSP Violation:", {
      violatedDirective: report["violated-directive"],
      blockedURI: report["blocked-uri"],
      documentURI: report["document-uri"],
      sourceFile: report["source-file"],
      lineNumber: report["line-number"],
    });
    res.status(204).end();
  },
);
```

### Testing CSP Without Enforcing

```http
# Use report-only mode to see what would be blocked without actually blocking
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /api/csp-report
# No resources blocked, but violations are reported
# Use to validate CSP before enforcement
```

---

## 9. Subresource Integrity (SRI)

SRI ensures third-party scripts and stylesheets haven't been tampered with:

```html
<!-- Generate hash: openssl dgst -sha384 -binary file.js | openssl base64 -A -->

<!-- Script with SRI: browser verifies hash before executing -->
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous"
></script>

<!-- If hash doesn't match: browser refuses to execute the script -->
<!-- Protects against: CDN compromise, man-in-the-middle injection -->

<!-- CSS with SRI -->
<link
  rel="stylesheet"
  href="https://cdn.example.com/styles.css"
  integrity="sha384-..."
  crossorigin="anonymous"
/>
```

### Generating SRI Hashes

```bash
# Generate SHA-384 hash for a file
cat app.js | openssl dgst -sha384 -binary | openssl base64 -A

# Or using npm:
echo "sha384-$(cat file.js | openssl dgst -sha384 -binary | base64 -w 0)"
```

```javascript
// Build tool: automatically generate SRI hashes
// vite-plugin-html or webpack-subresource-integrity

// webpack.config.js:
const SriPlugin = require("webpack-subresource-integrity");

module.exports = {
  output: {
    crossOriginLoading: "anonymous",
  },
  plugins: [
    new SriPlugin({
      hashFuncNames: ["sha384"],
      enabled: process.env.NODE_ENV === "production",
    }),
  ],
};
```

---

## 10. Mixed Content

Mixed content occurs when an HTTPS page loads resources over HTTP:

```
MIXED CONTENT TYPES:

Active mixed content (BLOCKED by default in all modern browsers):
  Script:  <script src="http://...">     → blocked, can hijack page
  Iframe:  <iframe src="http://...">     → blocked
  CSS:     <link href="http://...">      → blocked
  XHR/Fetch to http://...               → blocked

Passive mixed content (DEPRECATED warning, may be blocked):
  Images: <img src="http://...">         → blocked in most browsers
  Audio/Video: <source src="http://...">

All mixed content in 2024+: blocked by Chrome, Firefox, Safari
```

### Fixing Mixed Content

```html
<!-- ❌ HTTP resource on HTTPS page -->
<img src="http://cdn.example.com/photo.jpg" />

<!-- ✅ Use HTTPS explicitly -->
<img src="https://cdn.example.com/photo.jpg" />

<!-- ✅ Use protocol-relative URL (inherits page's protocol) -->
<img src="//cdn.example.com/photo.jpg" />
<!-- Note: protocol-relative URLs are generally discouraged;
     prefer explicit https: -->
```

```http
# Automatically upgrade HTTP requests to HTTPS:
Content-Security-Policy: upgrade-insecure-requests
# All http:// sub-resource requests are automatically changed to https://

# Block all mixed content (strict):
Content-Security-Policy: block-all-mixed-content
```

---

## 11. Certificate Transparency and HSTS

### HTTP Strict Transport Security (HSTS)

```http
# Tell browser: always use HTTPS for this domain, never HTTP
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

# max-age=31536000: remember for 1 year
# includeSubDomains: apply to all subdomains
# preload: submit to browser preload lists (hardcoded HTTPS-only in browsers)

# After receiving HSTS: browser NEVER makes HTTP request to this domain
# → Protects against SSL stripping attacks
# → User typing "example.com" → browser silently upgrades to https://
```

### HSTS Preloading

```
Submit to https://hstspreload.org for inclusion in browser preload lists

Requirements:
  - HTTPS available
  - max-age >= 31536000 (1 year)
  - includeSubDomains
  - preload directive
  - All subdomains also support HTTPS

Result: domain hardcoded in Chrome, Firefox, Safari as HTTPS-only
Even first visit: no HTTP → HTTPS redirect (can't be downgraded)
```

### Certificate Pinning (Public Key Pinning)

```
HTTP Public Key Pinning (HPKP) is deprecated.
Use Certificate Transparency monitoring instead.

HPKP was deprecated because:
  Wrong pin or lost key = site permanently inaccessible
  High risk of DoS by misconfiguration

Certificate Transparency (CT):
  All CAs must log certificates to public logs
  Browsers verify certificates appear in CT logs
  If a CA issues a fraudulent cert: detectable via CT logs

Expect-CT header (deprecated in 2021, CT is now mandatory):
Expect-CT: enforce, max-age=86400, report-uri="https://report.example.com/ct"
```

---

## 12. Good Practices

### ✅ Specific origin whitelist over wildcard

```javascript
// ✅ Never use * with credentials, always use specific origins
const CORS_WHITELIST = new Set([
  "https://app.example.com",
  "https://staging.example.com",
]);

function originValidator(origin, callback) {
  if (!origin || CORS_WHITELIST.has(origin)) {
    callback(null, true);
  } else {
    callback(new Error(`CORS: ${origin} not allowed`));
  }
}
```

### ✅ Always add Vary: Origin when reflecting dynamic origin

```javascript
// ✅ CDN will create separate cache entry per origin
res.setHeader("Access-Control-Allow-Origin", reflectedOrigin);
res.setHeader("Vary", "Origin"); // ALWAYS when dynamic
```

### ✅ Set CSP headers on all HTML responses

```javascript
// ✅ CSP as defense-in-depth against XSS
const cspMiddleware = (req, res, next) => {
  const nonce = crypto.randomBytes(16).toString("base64");
  res.locals.nonce = nonce;
  res.setHeader("Content-Security-Policy", buildCSP(nonce));
  next();
};
```

### ✅ Use SRI for third-party scripts

```html
<!-- ✅ Hash verified before execution -->
<script
  src="https://cdn.third-party.com/lib.js"
  integrity="sha384-..."
  crossorigin="anonymous"
></script>
```

---

## 13. Bad Practices

### ❌ `Access-Control-Allow-Origin: *` for authenticated APIs

```javascript
// ❌ Wildcard + auth endpoint = security hole
app.use(cors({ origin: "*" })); // all origins

app.get("/api/user/profile", authenticate, handler);
// GET /api/user/profile → returns user's private profile to ANY origin

// ✅ Restrict to specific origins for authenticated endpoints
app.get(
  "/api/user/profile",
  cors({ origin: ALLOWED_ORIGINS }),
  authenticate,
  handler,
);
```

### ❌ Disabling CORS by proxying everything through localhost

```javascript
// ❌ "Fix" CORS by routing all API calls through a local proxy
// vite.config.js proxy:
proxy: {
  '/api': 'https://third-party-api.com'  // all /api requests go to third-party
}
// This works in development but:
// - Exposes third-party API keys in browser
// - Doesn't scale to production
// - Hides real cross-origin issues

// ✅ In production: proper CORS configuration or a backend proxy
// ✅ In development: proxy is fine, but document that it's dev-only
```

### ❌ Ignoring CSP violations

```javascript
// ❌ Setting up CSP reporting but never reading the reports
// Violations indicate: attempted XSS, misconfigured CSP, broken third-party scripts

// ✅ Alert on high-severity CSP violations:
app.post("/api/csp-report", (req, res) => {
  const { "violated-directive": dir, "blocked-uri": uri } =
    req.body["csp-report"];

  if (dir.startsWith("script-src")) {
    // Script CSP violation: possible XSS attempt
    alertSecurityTeam({ dir, uri, timestamp: Date.now() });
  }

  metrics.increment("csp_violation", { directive: dir });
  res.status(204).end();
});
```

---

## 14. Common Mistakes

### Mistake 1 — CORS won't protect server-side resources from direct requests

```javascript
// CORS protects what BROWSER JavaScript can read — not what CAN be requested
// Server still processes ALL requests, with or without CORS headers

// ❌ Thinking CORS = API security
// Adding CORS headers does NOT protect your API from:
//   - curl requests
//   - Postman
//   - Backend-to-backend calls
//   - Malicious server-side code

// ✅ API security requires: authentication + authorization (not CORS)
// CORS is purely about what browser JavaScript can read
```

### Mistake 2 — Forgetting CORS for preflight OPTIONS requests

```javascript
// ❌ Auth middleware blocks OPTIONS requests
app.use(authenticate); // runs BEFORE OPTIONS handler
app.put("/api/data", handler);

// Browser sends: OPTIONS /api/data → 401 Unauthorized
// Actual PUT request never sent!

// ✅ Skip auth for OPTIONS preflight
app.use((req, res, next) => {
  if (req.method === "OPTIONS") return next(); // skip auth for preflight
  authenticate(req, res, next);
});
```

### Mistake 3 — Not setting CORS headers on error responses

```javascript
// ❌ CORS headers only on success responses
app.get("/api/data", (req, res, next) => {
  try {
    const data = getRequiredData();
    res.set("Access-Control-Allow-Origin", origin);
    res.json(data);
  } catch (err) {
    res.status(500).json({ error: err.message });
    // CORS header NOT set on 500 response!
    // Browser sees: 500 response without CORS header → network error!
    // Developer confused: "why is my API returning network errors?"
  }
});

// ✅ CORS headers on ALL responses (middleware approach)
app.use(cors(corsOptions)); // applies to ALL responses including errors
```

---

## 15. Interview-Level Explanation

> **"What is CORS? How does it work, and how do you configure it correctly?"**

**Strong answer:**

> "CORS — Cross-Origin Resource Sharing — is the mechanism that allows servers to tell browsers which origins are permitted to read their responses. It extends the Same-Origin Policy, which by default prevents JavaScript on one origin from reading HTTP responses from another origin.
>
> The key insight that trips people up: CORS is enforced by browsers, not servers. The server always processes the request. CORS only controls whether the browser will give the response to the requesting JavaScript. This means CORS provides no protection against direct API calls from curl, backend services, or Postman — it only protects against unauthorized cross-origin browser JavaScript.
>
> There are two types of requests. Simple requests — GET, HEAD, POST with specific content types — go directly to the server. The browser checks the response for `Access-Control-Allow-Origin` and either passes the response to JavaScript or blocks it. Preflighted requests — any PUT/DELETE/PATCH, or requests with JSON content-type or custom headers — trigger an OPTIONS preflight first. The browser sends an OPTIONS request asking if this cross-origin request is permitted, and only proceeds with the actual request if the server says yes. Preflight responses can be cached with `Access-Control-Max-Age` to avoid the round trip on subsequent requests.
>
> The most common security mistake is reflecting any `Origin` header back without validation: `Access-Control-Allow-Origin: req.headers.origin`. Combined with `Access-Control-Allow-Credentials: true`, this lets any website read your authenticated API responses. The fix is a strict whitelist — only reflect origins that are in your allowlist, and always set `Vary: Origin` when you do, so CDNs don't cache the wrong origin in the CORS header.
>
> The wildcard `*` can't be used with credentials. If you need cookies or Authorization headers to work cross-origin, you must specify the exact origin and set `Access-Control-Allow-Credentials: true`. For public APIs that don't need credentials, `*` is fine and simplest.
>
> For comprehensive web security beyond CORS: Content Security Policy restricts which resources can be loaded, protecting against XSS. Subresource Integrity verifies third-party script hashes before execution. HSTS forces HTTPS. These work together as defense-in-depth."

---

## 16. Exercises

### Exercise 1 — CORS Configuration Analysis

```http
# Review this server CORS configuration and identify all issues:

Request:
GET /api/user/data HTTP/1.1
Origin: https://evil.com
Cookie: session=abc123

Response:
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.com   ← reflects any origin
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
Access-Control-Allow-Headers: *
Content-Type: application/json

{"userId": "42", "email": "victim@example.com", "creditCard": "4111..."}
```

<details>
<summary>Answer</summary>

```
Issues found:

1. CRITICAL: Reflects any Origin with Allow-Credentials: true
   Access-Control-Allow-Origin: https://evil.com (reflects request origin)
   Access-Control-Allow-Credentials: true

   This allows evil.com's JavaScript to:
   - Make requests to your API with the victim's session cookie
   - Read the full response (user data, credit card info)
   - This is a textbook CORS credential exposure vulnerability

   Fix: Whitelist only trusted origins:
   if (ALLOWED_ORIGINS.has(origin)) {
     res.setHeader('Access-Control-Allow-Origin', origin);
     res.setHeader('Vary', 'Origin');  // ← add Vary
   }

2. Access-Control-Allow-Headers: *
   Wildcard for headers cannot be used with credentials
   Browser will reject this combination

   Fix: List specific allowed headers:
   Access-Control-Allow-Headers: Content-Type, Authorization

3. No Vary: Origin header
   Without Vary: Origin, CDN may cache response with evil.com's ACAO
   and serve it to legitimate origins

   Fix: Always add Vary: Origin when reflecting dynamic origin

4. Sensitive data (credit card) in GET response
   GET responses may be cached (by browser, CDN, proxy)
   Should be:
   - POST/not cacheable
   - Cache-Control: no-store
   - Require explicit scope/permission for accessing payment data

5. ACAO: * would not work with credentials anyway
   If the server intends public access: use * without credentials
   If it needs credentials: use specific origin whitelist
   Cannot combine: ACAO: * with ACAC: true
```

</details>

---

## 🔗 Related Topics

- [`networking/02-fetch-and-xhr.md`](./02-fetch-and-xhr.md) — Fetch CORS mode and credentials
- [`security/01-xss.md`](../security/01-xss.md) — CSP protects against XSS
- [`security/02-csrf.md`](../security/02-csrf.md) — CORS and CSRF relationship
- [`caching/01-http-caching.md`](../caching/01-http-caching.md) — Vary header for CDN caching

---

<div align="center">

**`networking/` section complete!** 🎉

All 4 networking files done:
`01-http-protocols.md` · `02-fetch-and-xhr.md` · `03-websockets-sse.md` · **`04-cors-and-security.md`** ✓

**Next section:** [`security/`](../security/) →

</div>
