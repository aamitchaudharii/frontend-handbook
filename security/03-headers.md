# 03 — Security Headers

> **"Security headers are the fastest return on security investment. Adding six response headers takes 30 minutes and defends against XSS, clickjacking, MIME sniffing, information disclosure, and protocol downgrade attacks. Most applications serve millions of requests without these defenses simply because nobody added them."**

HTTP security headers are response headers that instruct the browser to apply security policies to the current page and its resources. They're a critical layer of defense-in-depth: even if an attacker finds a vulnerability in your application code, properly configured security headers can prevent that vulnerability from being exploited, limit its blast radius, or eliminate entire attack categories. This document covers every major security header, their directives, misconfiguration risks, and the configuration patterns for common server environments.

---

## 📚 Table of Contents

1. [Content-Security-Policy (CSP)](#1-content-security-policy-csp)
2. [Strict-Transport-Security (HSTS)](#2-strict-transport-security-hsts)
3. [X-Frame-Options](#3-x-frame-options)
4. [X-Content-Type-Options](#4-x-content-type-options)
5. [Referrer-Policy](#5-referrer-policy)
6. [Permissions-Policy](#6-permissions-policy)
7. [Cross-Origin Headers (COEP, COOP, CORP)](#7-cross-origin-headers-coep-coop-corp)
8. [Cache-Control for Security](#8-cache-control-for-security)
9. [Server Header and Information Disclosure](#9-server-header-and-information-disclosure)
10. [Security Headers Audit](#10-security-headers-audit)
11. [Server Configuration Examples](#11-server-configuration-examples)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. Content-Security-Policy (CSP)

CSP is the most powerful security header. It defines the sources from which resources may be loaded and executes scripts.

### Core Directives

```http
Content-Security-Policy:
  # Fallback for directives not explicitly set
  default-src 'self';

  # Script sources (most critical)
  script-src 'self' 'nonce-{NONCE}' https://cdn.trusted.com;

  # Style sources
  style-src 'self' 'nonce-{NONCE}' https://fonts.googleapis.com;

  # Image sources
  img-src 'self' data: https://cdn.example.com;

  # Font sources
  font-src 'self' https://fonts.gstatic.com;

  # Fetch/XHR/WebSocket destinations
  connect-src 'self' https://api.example.com wss://ws.example.com;

  # iframe sources (where you CAN embed iframes from)
  frame-src 'none';

  # Where your page CAN BE embedded (replaces X-Frame-Options)
  frame-ancestors 'none';

  # Plugin sources
  object-src 'none';

  # Web Worker sources
  worker-src 'self';

  # Service Worker sources
  manifest-src 'self';

  # Media sources (audio, video)
  media-src 'self' https://media.example.com;

  # Restricts where forms can submit
  form-action 'self';

  # Restricts <base> href
  base-uri 'self';

  # Force HTTP to HTTPS for all sub-resource requests
  upgrade-insecure-requests;

  # Violation reporting
  report-uri /api/csp-report;
  report-to csp-violations;
```

### CSP Source Values

```
'self'              Same origin as the page
'none'              No sources (blocks everything for this directive)
'unsafe-inline'     Inline scripts/styles (defeats XSS protection — avoid!)
'unsafe-eval'       eval(), new Function() (avoid!)
'strict-dynamic'    Trust nonces/hashes; trusted scripts can load other scripts
'unsafe-hashes'     Allow specific inline event handlers by hash
https:              Any HTTPS source (too broad)
data:               data: URIs (needed for inline images/fonts)
blob:               Blob URLs
wss:                WebSocket Secure (for connect-src)
'nonce-{base64}'    Script/style with specific nonce attribute
'sha256-{hash}'     Script/style matching this specific hash
example.com         A specific domain (all schemes)
https://example.com A specific domain, HTTPS only
*.example.com       Any subdomain of example.com
```

### Strict CSP (Nonce-Based)

```javascript
// The recommended modern approach: nonce-based, no unsafe-inline
// Generate fresh nonce per request
const nonce = crypto.randomUUID().replace(/-/g, "").slice(0, 22);
const cspBase64 = Buffer.from(nonce).toString("base64");

res.setHeader(
  "Content-Security-Policy",
  [
    "default-src 'self'",
    `script-src 'self' 'nonce-${cspBase64}' 'strict-dynamic'`,
    // strict-dynamic: trusted scripts (with nonce) can load other scripts
    // This allows bundlers that inject script tags dynamically
    `style-src 'self' 'nonce-${cspBase64}'`,
    "img-src 'self' data: https:",
    "connect-src 'self' https://api.example.com",
    "font-src 'self' https://fonts.gstatic.com",
    "frame-ancestors 'none'",
    "object-src 'none'",
    "base-uri 'self'",
    "upgrade-insecure-requests",
    "report-to csp-violations",
  ].join("; "),
);

// HTML template (server injects nonce):
// <script nonce="${cspBase64}" src="/app.js"></script>
```

### Hash-Based CSP (for Static Sites)

```javascript
// When you can't use nonces (static HTML, CDN-served pages):
// Pre-compute SHA-256 hashes of inline scripts

const script = `console.log('hello');`;
const hash = crypto.createHash("sha256").update(script).digest("base64");
// hash = "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU="

// In HTML:
// <script>console.log('hello');</script>

// CSP header:
`script-src 'self' 'sha256-${hash}'`;
// Only this exact inline script is allowed; any other inline script is blocked
```

### CSP Reporting

```javascript
// Report-To header (modern replacement for report-uri)
res.setHeader(
  "Report-To",
  JSON.stringify({
    group: "csp-violations",
    max_age: 86400,
    endpoints: [{ url: "https://report.example.com/csp" }],
  }),
);

res.setHeader(
  "Content-Security-Policy",
  `default-src 'self'; report-to csp-violations`,
);

// Collect reports on your server
app.post(
  "/api/csp-report",
  express.json({ type: ["application/csp-report", "application/json"] }),
  (req, res) => {
    const report = req.body["csp-report"] || req.body;

    // Log or alert on high-severity violations
    if (report["violated-directive"]?.startsWith("script-src")) {
      securityAlert.send({
        type: "CSP_VIOLATION",
        directive: report["violated-directive"],
        blocked: report["blocked-uri"],
        page: report["document-uri"],
      });
    }

    metrics.increment("csp.violation", {
      directive: report["violated-directive"],
    });

    res.status(204).end();
  },
);
```

---

## 2. Strict-Transport-Security (HSTS)

Forces browsers to use HTTPS for all future requests:

```http
# Basic HSTS: 1 year max-age
Strict-Transport-Security: max-age=31536000

# Include all subdomains
Strict-Transport-Security: max-age=31536000; includeSubDomains

# Enable preloading (requires submission to browser preload lists)
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

### HSTS Behavior

```
FIRST VISIT (no HSTS yet):
  User types: example.com
  Browser: tries http://example.com → redirect to https:// → follows redirect
  Server: sends HSTS header
  Browser stores: "example.com uses HTTPS only for 31536000 seconds"

SUBSEQUENT VISITS (HSTS active):
  User types: example.com
  Browser: internally rewrites to https://example.com (no HTTP request at all)
  No HTTP request → no plaintext exposure → SSL stripping impossible

HSTS PRELOAD:
  Domain submitted to https://hstspreload.org
  Included in Chrome/Firefox/Safari source code
  Even FIRST visit: browser knows to use HTTPS (hardcoded)
```

### HSTS Pitfalls

```javascript
// ❌ Setting HSTS on HTTP responses (ignored by browsers for HTTPS enforcement)
// HTTP response with HSTS header is ignored

// ❌ Short max-age (provides minimal protection)
Strict-Transport-Security: max-age=300  // only 5 minutes

// ❌ HSTS without valid HTTPS cert on all subdomains (with includeSubDomains)
// If dev.example.com has no HTTPS cert but HSTS includes subdomains:
// dev.example.com becomes inaccessible for the entire max-age period

// ✅ Build up gradually:
// Phase 1: max-age=300 (5 min) — test that nothing breaks
// Phase 2: max-age=86400 (1 day)
// Phase 3: max-age=604800 (1 week)
// Phase 4: max-age=31536000; includeSubDomains
// Phase 5: add preload (permanent commitment!)
```

---

## 3. X-Frame-Options

Prevents your page from being embedded in an iframe (clickjacking defense):

```http
# Most restrictive: no iframes at all
X-Frame-Options: DENY

# Allow same-origin iframes only
X-Frame-Options: SAMEORIGIN

# Deprecated: allow specific origin (inconsistent browser support)
X-Frame-Options: ALLOW-FROM https://trusted.example.com
```

### Clickjacking Attack

```html
<!-- Attacker overlays victim.com in a transparent iframe -->
<style>
  #victim { position: absolute; top: 0; left: 0; width: 100%; height: 100%; opacity: 0; }
  #decoy  { position: relative; } /* the visible page with a "win free stuff" button -->
</style>
<iframe
  id="victim"
  src="https://victim.com/transfer?to=attacker&amount=1000"
></iframe>
<div id="decoy">
  <button style="position:absolute; top: 300px; left: 200px;">
    Claim Your Prize!
  </button>
  <!-- Button is positioned exactly over victim.com's "Confirm" button -->
  <!-- User clicks the prize button → actually clicks confirm transfer -->
</div>
```

### X-Frame-Options vs CSP frame-ancestors

```
X-Frame-Options: older header, widely supported, simple
  DENY → no framing
  SAMEORIGIN → same origin only
  ALLOW-FROM → inconsistent browser support, avoid

CSP frame-ancestors: modern replacement, more powerful
  'none'              → no framing (equivalent to DENY)
  'self'              → same origin only (equivalent to SAMEORIGIN)
  https://trusted.com → specific trusted framer
  *.example.com       → any subdomain can frame it

RECOMMENDATION: set BOTH for maximum compatibility
  X-Frame-Options: DENY
  Content-Security-Policy: frame-ancestors 'none'
```

---

## 4. X-Content-Type-Options

Prevents MIME-type sniffing attacks:

```http
X-Content-Type-Options: nosniff
```

### MIME Sniffing Attack

```
ATTACK SCENARIO:
  Site allows file uploads, user uploads "image.jpg" containing JavaScript
  Server sets: Content-Type: image/jpeg
  Without nosniff: IE sniffs the content, detects JavaScript, executes it!
  With nosniff: browser respects Content-Type header → treats as image → safe

MECHANISM:
  Without nosniff: browsers analyze content to detect "real" MIME type
  With nosniff: browser strictly respects the Content-Type header

PROTECTION:
  Prevents: JavaScript execution from files served as other types
  Prevents: CSS execution from files served as other types

SET ON ALL RESPONSES: especially file download endpoints
```

---

## 5. Referrer-Policy

Controls what URL information is sent in the `Referer` header:

```http
# Send no Referer header for any request
Referrer-Policy: no-referrer

# Send Referer only for same-origin requests
Referrer-Policy: same-origin

# Send origin only (no path/query) for cross-origin HTTPS
# Send full URL for same-origin (recommended)
Referrer-Policy: strict-origin-when-cross-origin

# Send origin only for all requests (no path/query)
Referrer-Policy: origin

# Send full URL (path + query) for same-origin and HTTPS cross-origin
# Send origin only for HTTP cross-origin
Referrer-Policy: no-referrer-when-downgrade (browser default, but may leak data)
```

### Why Referrer-Policy Matters

```
WITHOUT POLICY (no-referrer-when-downgrade):
  User on: https://example.com/sensitive-document?user_id=42&token=secret
  Clicks link to: https://third-party.com
  Referer header: https://example.com/sensitive-document?user_id=42&token=secret
  → Third-party sees user_id, token, document path!

WITH strict-origin-when-cross-origin:
  Referer header: https://example.com
  → Third-party sees only the domain, no sensitive path/query data

PRACTICAL IMPACT:
  User IDs in URLs: protected
  Session tokens in URLs: protected (don't put tokens in URLs anyway!)
  Health/medical page names: protected from third-party analytics
  Password reset tokens in URLs: protected
```

---

## 6. Permissions-Policy

Controls access to browser features (formerly Feature-Policy):

```http
# Disable powerful features unless explicitly needed
Permissions-Policy:
  # Geolocation
  geolocation=(),

  # Microphone
  microphone=(),

  # Camera
  camera=(),

  # Fullscreen (allow self only)
  fullscreen=(self),

  # Payment (allow self + trusted payment domain)
  payment=(self "https://checkout.stripe.com"),

  # USB access (deny all)
  usb=(),

  # Accelerometer / Gyroscope (useful for analytics abuse)
  accelerometer=(),
  gyroscope=(),

  # Autoplay
  autoplay=(self)
```

### Why Permissions-Policy Matters

```
SCENARIO WITHOUT POLICY:
  Third-party script included from ads.example.com
  Script calls: navigator.geolocation.getCurrentPosition(sendToServer)
  → Third party reads user's location via your page

WITH Permissions-Policy: geolocation=():
  navigator.geolocation.getCurrentPosition → SecurityError
  → Third party cannot access geolocation through your page

PARTICULARLY IMPORTANT FOR:
  Sites with third-party scripts (analytics, ads, widgets)
  Sites with iframes embedding third-party content
  Sites handling sensitive user interactions
```

---

## 7. Cross-Origin Headers (COEP, COOP, CORP)

Three headers that together enable powerful browser features (SharedArrayBuffer, high-resolution timers) while protecting against Spectre-class attacks:

```http
# Cross-Origin-Embedder-Policy: require CORP on all sub-resources
Cross-Origin-Embedder-Policy: require-corp
# OR
Cross-Origin-Embedder-Policy: credentialless

# Cross-Origin-Opener-Policy: isolate window from cross-origin openers
Cross-Origin-Opener-Policy: same-origin

# Cross-Origin-Resource-Policy: control who can embed this resource
Cross-Origin-Resource-Policy: same-origin      # only same-origin pages
Cross-Origin-Resource-Policy: same-site        # same-site pages
Cross-Origin-Resource-Policy: cross-origin     # anyone (use for CDN assets)
```

### Why These Headers Exist

```
SPECTRE ATTACK (2018):
  Speculative execution side-channel
  JavaScript can potentially read other processes' memory via timing
  → Could read cross-origin data (cookies, passwords, etc.)

BROWSER MITIGATION:
  High-resolution timers (SharedArrayBuffer enables) made Spectre practical
  Browsers removed SharedArrayBuffer in 2018

TO RE-ENABLE SharedArrayBuffer (needed for WebAssembly, video editing, etc.):
  Must opt into cross-origin isolation:
  Cross-Origin-Embedder-Policy: require-corp
  Cross-Origin-Opener-Policy: same-origin

  Result: Cross-origin isolation
  window.crossOriginIsolated === true
  SharedArrayBuffer: re-enabled
  But: all resources on page must opt in via CORP

PRACTICAL IMPACT:
  If you need SharedArrayBuffer: implement COEP + COOP
  If your CDN resources need to be embedded by others:
    Set Cross-Origin-Resource-Policy: cross-origin on CDN responses
    (Otherwise COEP blocks them on pages that use require-corp)
```

```javascript
// Checking cross-origin isolation status:
if (window.crossOriginIsolated) {
  // SharedArrayBuffer available
  const buffer = new SharedArrayBuffer(1024);
  const worker = new Worker("worker.js");
  worker.postMessage(buffer, [buffer]);
} else {
  // Fallback path without SharedArrayBuffer
}
```

---

## 8. Cache-Control for Security

Certain pages should never be cached to prevent sensitive data exposure:

```http
# Sensitive pages: never cache
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache  # Legacy HTTP/1.0 support
Expires: 0        # Legacy

# Use for: login pages, account pages, payment pages, admin panels
# Prevents: back-button access to authenticated pages after logout
#           proxy/CDN caching of personal data
#           browser auto-fill from cached forms

# Example: post-logout redirect target
app.get('/account', authenticate, (req, res) => {
  res.setHeader('Cache-Control', 'no-store');
  res.render('account', { user: req.user });
});
```

---

## 9. Server Header and Information Disclosure

Remove headers that reveal server technology — reduce attacker reconnaissance:

```http
# ❌ Default nginx: reveals server and version
Server: nginx/1.18.0

# ❌ Default Express: reveals technology
X-Powered-By: Express

# ❌ PHP: reveals version
X-Powered-By: PHP/8.1.12
```

```javascript
// Express: remove X-Powered-By
app.disable('x-powered-by');

// Or with helmet:
app.use(helmet({ hidePoweredBy: true }));

// nginx: suppress server version
server_tokens off;  # shows "nginx" without version

# Apache: suppress server info
ServerTokens Prod  # shows "Apache" without version
ServerSignature Off
```

---

## 10. Security Headers Audit

### Quick Audit Script

```javascript
// Check your site's security headers
async function auditSecurityHeaders(url) {
  const response = await fetch(url, { method: "HEAD", redirect: "follow" });
  const headers = Object.fromEntries(response.headers.entries());

  const checks = {
    "Content-Security-Policy": {
      present: "csp-header" in headers || "content-security-policy" in headers,
      required: true,
      severity: "critical",
    },
    "Strict-Transport-Security": {
      present: "strict-transport-security" in headers,
      required: url.startsWith("https"),
      severity: "high",
    },
    "X-Frame-Options": {
      present: "x-frame-options" in headers,
      required: true,
      severity: "medium",
    },
    "X-Content-Type-Options": {
      present: headers["x-content-type-options"] === "nosniff",
      required: true,
      severity: "medium",
    },
    "Referrer-Policy": {
      present: "referrer-policy" in headers,
      required: true,
      severity: "low",
    },
    "Permissions-Policy": {
      present: "permissions-policy" in headers,
      required: false,
      severity: "low",
    },
  };

  Object.entries(checks).forEach(([header, check]) => {
    const status = check.present
      ? "✓"
      : check.required
        ? "✗ MISSING"
        : "- Optional";
    console.log(`${status} ${header} (${check.severity})`);
  });
}

// Online tools:
// https://securityheaders.com
// https://observatory.mozilla.org
```

### Security Score Targets

```
A+ (securityheaders.com score):
  ✓ CSP with no unsafe-inline/unsafe-eval
  ✓ HSTS with includeSubDomains and preload
  ✓ X-Frame-Options: DENY
  ✓ X-Content-Type-Options: nosniff
  ✓ Referrer-Policy (non-default)
  ✓ Permissions-Policy
  ✓ COEP + COOP (if using advanced features)
```

---

## 11. Server Configuration Examples

### Express.js with Helmet

```javascript
import helmet from "helmet";
import crypto from "crypto";

app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString("base64");
  next();
});

app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: [
          "'self'",
          (req, res) => `'nonce-${res.locals.nonce}'`,
          "'strict-dynamic'",
        ],
        styleSrc: ["'self'", (req, res) => `'nonce-${res.locals.nonce}'`],
        imgSrc: ["'self'", "data:", "https:"],
        connectSrc: ["'self'", "https://api.example.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        frameSrc: ["'none'"],
        frameAncestors: ["'none'"],
        objectSrc: ["'none'"],
        baseUri: ["'self'"],
        formAction: ["'self'"],
        upgradeInsecureRequests: [],
      },
      reportOnly: false, // enforce (true = report-only mode)
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
    referrerPolicy: { policy: "strict-origin-when-cross-origin" },
    crossOriginEmbedderPolicy: { policy: "require-corp" },
    crossOriginOpenerPolicy: { policy: "same-origin" },
    crossOriginResourcePolicy: { policy: "same-origin" },
    permissionsPolicy: {
      features: {
        geolocation: [], // disabled
        microphone: [], // disabled
        camera: [], // disabled
        fullscreen: ["'self'"],
        payment: ["'self'"],
      },
    },
  }),
);
```

### Nginx Configuration

```nginx
# nginx security headers
server {
  # HSTS
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

  # Content Security Policy
  add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'nonce-$request_id'; style-src 'self' 'nonce-$request_id'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; object-src 'none'; base-uri 'self'; upgrade-insecure-requests" always;

  # Clickjacking
  add_header X-Frame-Options "DENY" always;

  # MIME sniffing
  add_header X-Content-Type-Options "nosniff" always;

  # Referrer
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;

  # Permissions
  add_header Permissions-Policy "geolocation=(), microphone=(), camera=(), usb=(), payment=(self)" always;

  # Server info
  server_tokens off;
}
```

### Next.js Configuration

```javascript
// next.config.js
const securityHeaders = [
  {
    key: "Strict-Transport-Security",
    value: "max-age=31536000; includeSubDomains; preload",
  },
  {
    key: "X-Frame-Options",
    value: "DENY",
  },
  {
    key: "X-Content-Type-Options",
    value: "nosniff",
  },
  {
    key: "Referrer-Policy",
    value: "strict-origin-when-cross-origin",
  },
  {
    key: "Permissions-Policy",
    value: "geolocation=(), microphone=(), camera=()",
  },
];

module.exports = {
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: securityHeaders,
      },
    ];
  },
};
```

---

## 12. Good Practices

### ✅ Apply headers to ALL responses, not just HTML

```javascript
// ✅ Headers on every response (use `always` in nginx)
// X-Content-Type-Options: nosniff on JSON/binary responses too
// Prevents MIME sniffing of API responses

// ✅ HSTS only on HTTPS responses (browser ignores it on HTTP)
// ✅ CSP can be tuned per response type (stricter for HTML, looser for assets)
```

### ✅ Use report-only mode before enforcing CSP

```http
# Phase 1: Monitor what CSP would block (no breakage)
Content-Security-Policy-Report-Only: default-src 'self'; script-src 'self'; report-uri /csp-report

# Phase 2: After fixing violations, enforce
Content-Security-Policy: default-src 'self'; script-src 'self'; report-uri /csp-report
```

### ✅ Automate security header checks in CI

```javascript
// Security header test in Jest:
describe("Security Headers", () => {
  let response;
  beforeAll(async () => {
    response = await fetch("http://localhost:3000");
  });

  test("HSTS header present", () => {
    expect(response.headers.get("Strict-Transport-Security")).toMatch(
      /max-age=\d+/,
    );
  });

  test("CSP header present", () => {
    expect(response.headers.get("Content-Security-Policy")).toBeTruthy();
  });

  test("X-Content-Type-Options is nosniff", () => {
    expect(response.headers.get("X-Content-Type-Options")).toBe("nosniff");
  });

  test("No X-Powered-By disclosure", () => {
    expect(response.headers.get("X-Powered-By")).toBeNull();
  });
});
```

---

## 13. Bad Practices

### ❌ CSP with `unsafe-inline` defeats XSS protection

```http
# ❌ unsafe-inline negates most XSS protection
Content-Security-Policy: script-src 'self' 'unsafe-inline'
# An attacker who injects <script>evil()</script> can execute it
# Because 'unsafe-inline' allows ANY inline script

# ✅ Use nonces or hashes instead
Content-Security-Policy: script-src 'self' 'nonce-{RANDOM}'
```

### ❌ HSTS without valid HTTPS on all subdomains (with includeSubDomains)

```
❌ Setting includeSubDomains when test.example.com has no HTTPS cert
   Result: test.example.com unreachable for max-age period (1 year!)
   Users get HSTS errors trying to access test subdomain

✅ Only add includeSubDomains when ALL subdomains serve valid HTTPS
```

### ❌ Permissive CSP to avoid breakage

```http
# ❌ CSP so permissive it provides no protection
Content-Security-Policy: default-src *; script-src * 'unsafe-inline' 'unsafe-eval'
# This is essentially no CSP — worse than having no header (false sense of security)
```

---

## 14. Common Mistakes

### Mistake 1 — Missing `always` in nginx for error responses

```nginx
# ❌ Headers only on 2xx responses
add_header X-Frame-Options "DENY";
# 404, 500 pages: no security headers!
# Attacker can iframe an error page

# ✅ always keyword: adds header to ALL responses
add_header X-Frame-Options "DENY" always;
```

### Mistake 2 — CSP blocking legitimate resources in production

```javascript
// Start with report-only to catch issues before enforcing
// Monitor reports for 1-2 weeks before switching to enforce

// Common CSP violations to prepare for:
// - Google Tag Manager (inline scripts)
// - Third-party chat widgets
// - Fonts from external CDNs
// - Analytics tracking pixels
// - Browser extensions injecting scripts (ignore these)

// Solution: whitelist specific domains and use nonces for inline scripts
```

### Mistake 3 — Inconsistent headers between development and production

```javascript
// ❌ Security headers only in production
if (process.env.NODE_ENV === "production") {
  app.use(helmet());
}
// Result: developers never test with CSP enabled
// CSP violations discovered only in production

// ✅ Same headers in all environments (with report-only in dev)
const cspMode =
  process.env.CSP_ENFORCE === "true"
    ? "Content-Security-Policy"
    : "Content-Security-Policy-Report-Only";

app.use((req, res, next) => {
  res.setHeader(cspMode, buildCSP(res.locals.nonce));
  next();
});
```

---

## 15. Interview-Level Explanation

> **"What security headers should every web application set, and what do they protect against?"**

**Strong answer:**

> "I'd cover six essential security headers and what each defends against.
>
> Content Security Policy is the most important. It tells the browser where scripts, styles, images, and other resources may be loaded from. A strict CSP — one that uses nonces for scripts rather than `unsafe-inline` — prevents XSS from executing even if an attacker manages to inject HTML. The nonce approach generates a random value per request, embeds it in the CSP header and in allowed script tags, and the browser only executes scripts that match the nonce. Any injected script tags have no nonce and are blocked.
>
> Strict-Transport-Security enforces HTTPS. After the browser receives the HSTS header, it refuses to make HTTP connections to that domain for the max-age period — typically a year. This prevents SSL stripping attacks where a network attacker intercepts the first HTTP request. With HSTS preloading, even the first visit is protected.
>
> X-Frame-Options and its modern replacement `frame-ancestors` in CSP prevent clickjacking — embedding your page in a transparent iframe over a decoy page so users think they're clicking one thing but actually clicking elements on your page. Setting to DENY or 'none' prevents all iframing.
>
> X-Content-Type-Options: nosniff prevents MIME-type sniffing. Without it, older browsers would analyze response content to detect the 'real' type — so a file uploaded as an image but containing JavaScript could be executed. nosniff forces the browser to respect the Content-Type header.
>
> Referrer-Policy controls what URL information is included in the Referer header when users navigate away from your site. `strict-origin-when-cross-origin` sends only your domain to external sites, not the full path and query string — protecting sensitive data that may appear in URLs.
>
> Permissions-Policy restricts what browser APIs your page and its embedded third-party scripts can access. Disabling geolocation, microphone, and camera prevents malicious or compromised third-party scripts from covertly accessing these features.
>
> For implementation: Helmet.js for Node.js, or server-level configuration in nginx/Apache, makes this straightforward. The key discipline is testing CSP in report-only mode before enforcing it, since CSP often breaks legitimate third-party scripts and inline styles that weren't written with CSP in mind."

---

## 16. Exercises

### Exercise 1 — Build a security header configuration

Your application has these requirements:

- React SPA served by Express
- Embeds Google Analytics (external script)
- Uses Google Fonts (external CSS + fonts)
- Makes API calls to https://api.example.com
- Has a payment form that submits to https://stripe.com
- Must not be embeddable in iframes

Write the complete security header configuration using Express and helmet:

<details>
<summary>Solution</summary>

```javascript
import helmet from "helmet";
import crypto from "crypto";

// Nonce per request
app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString("base64");
  next();
});

app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],

        // Scripts: self, nonce, strict-dynamic (allows bundle to load chunks)
        // Google Analytics: tag.google.com
        scriptSrc: [
          "'self'",
          (req, res) => `'nonce-${res.locals.nonce}'`,
          "'strict-dynamic'",
          "https://www.googletagmanager.com",
          "https://www.google-analytics.com",
        ],

        // Styles: self, nonce, Google Fonts stylesheet
        styleSrc: [
          "'self'",
          (req, res) => `'nonce-${res.locals.nonce}'`,
          "https://fonts.googleapis.com",
        ],

        // Images: self, data URIs, Google Analytics tracking pixel
        imgSrc: [
          "'self'",
          "data:",
          "https://www.google-analytics.com",
          "https://www.googletagmanager.com",
        ],

        // Fonts: self + Google Fonts font files
        fontSrc: ["'self'", "https://fonts.gstatic.com"],

        // API calls: self + API domain + Google Analytics
        connectSrc: [
          "'self'",
          "https://api.example.com",
          "https://www.google-analytics.com",
          "https://analytics.google.com",
          "https://stats.g.doubleclick.net",
        ],

        // Forms: self + Stripe for payment
        formAction: [
          "'self'",
          "https://stripe.com",
          "https://checkout.stripe.com",
        ],

        // No iframes
        frameSrc: ["'none'"],
        frameAncestors: ["'none'"],

        // No plugins
        objectSrc: ["'none'"],

        // Restrict base href
        baseUri: ["'self'"],

        // Upgrade HTTP to HTTPS
        upgradeInsecureRequests: [],

        // Report violations
        reportTo: ["csp-violations"],
      },
    },

    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },

    referrerPolicy: {
      policy: "strict-origin-when-cross-origin",
    },

    crossOriginEmbedderPolicy: false, // disable if Google Analytics doesn't support CORP
    crossOriginOpenerPolicy: { policy: "same-origin-allow-popups" }, // Stripe needs popup
  }),
);

// Additional custom headers
app.use((req, res, next) => {
  res.setHeader(
    "Permissions-Policy",
    'geolocation=(), microphone=(), camera=(), payment=(self "https://stripe.com")',
  );
  res.setHeader(
    "Report-To",
    JSON.stringify({
      group: "csp-violations",
      max_age: 86400,
      endpoints: [{ url: "/api/csp-report" }],
    }),
  );
  next();
});
```

</details>

---

## 🔗 Related Topics

- [`security/01-xss.md`](./01-xss.md) — CSP as XSS defense
- [`security/02-csrf.md`](./02-csrf.md) — Cookie security headers
- [`networking/04-cors-and-security.md`](../networking/04-cors-and-security.md) — CORS + CSP relationship
- [`caching/01-http-caching.md`](../caching/01-http-caching.md) — Cache-Control for security

---

<div align="center">

**Next:** [`security/04-auth-patterns.md`](./04-auth-patterns.md) →

</div>
