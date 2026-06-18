# 02 — CSRF

> **"CSRF exploits something the browser does automatically and helpfully: it sends cookies on every matching request, whether the request originates from your site or from attacker.com. The attacker never needs to read the response — they just need the request to happen. That's what makes CSRF uniquely dangerous: the victim is the weapon."**

Cross-Site Request Forgery (CSRF) tricks an authenticated user into unknowingly submitting a malicious request to a site they're logged into. Unlike XSS, CSRF doesn't steal credentials — it weaponizes existing credentials. The victim's browser automatically attaches their session cookie to any request matching the target origin, so the attacker's page can force the victim to change their password, transfer money, or perform any action the victim is authorized to take. This document covers CSRF mechanics, the SameSite cookie attribute (the modern primary defense), token-based CSRF protection, and how modern web architecture affects the CSRF threat model.

---

## 📚 Table of Contents

1. [How CSRF Works](#1-how-csrf-works)
2. [What CSRF Can and Cannot Do](#2-what-csrf-can-and-cannot-do)
3. [CSRF Attack Vectors](#3-csrf-attack-vectors)
4. [Defense 1 — SameSite Cookie Attribute](#4-defense-1--samesite-cookie-attribute)
5. [Defense 2 — CSRF Tokens (Synchronizer Token Pattern)](#5-defense-2--csrf-tokens-synchronizer-token-pattern)
6. [Defense 3 — Double Submit Cookie](#6-defense-3--double-submit-cookie)
7. [Defense 4 — Origin / Referer Header Validation](#7-defense-4--origin--referer-header-validation)
8. [Defense 5 — Custom Request Headers](#8-defense-5--custom-request-headers)
9. [CSRF and Modern SPAs](#9-csrf-and-modern-spas)
10. [CSRF vs XSS](#10-csrf-vs-xss)
11. [Logout CSRF](#11-logout-csrf)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. How CSRF Works

### Step-by-Step Attack

```
SETUP:
  User is logged in to bank.com
  Their browser holds: Cookie: session=abc123 (valid session)

ATTACK:
  User visits attacker.com (e.g., via phishing link)
  attacker.com contains:
    <form action="https://bank.com/transfer" method="POST">
      <input type="hidden" name="to"     value="attacker-account">
      <input type="hidden" name="amount" value="10000">
    </form>
    <script>document.forms[0].submit();</script>

WHAT HAPPENS:
  Browser submits form to https://bank.com/transfer
  Browser automatically includes: Cookie: session=abc123
  bank.com: sees valid session cookie → processes the transfer!

  The user never saw a form — it submitted silently via JavaScript
  bank.com sent $10,000 to the attacker's account

KEY: the attacker never reads the response
     They only need the REQUEST to be sent with the victim's cookies
```

### Why the Browser Cooperates

```
Cookies are sent automatically by the browser based on:
  - The cookie's Domain attribute matches the request URL
  - The cookie's Path attribute matches
  - The cookie's Secure flag (HTTPS requirement)

The browser does NOT check:
  - Which page initiated the request
  - Whether JavaScript or HTML submitted the form

This automatic inclusion is a feature (enables SSO, authentication)
that CSRF exploits.
```

---

## 2. What CSRF Can and Cannot Do

```
CSRF CAN:
  ✓ Change user's email address or password
  ✓ Transfer money (if no secondary confirmation)
  ✓ Purchase items
  ✓ Delete accounts or data
  ✓ Post content as the user
  ✓ Change privacy settings
  ✓ Log the user out (logout CSRF)
  ✓ Modify any state-changing operation the user can perform

CSRF CANNOT:
  ✗ Read responses (cross-origin reads blocked by SOP)
  ✗ Steal session tokens from cookies (HttpOnly prevents this)
  ✗ Read sensitive data from the target site
  ✗ Bypass two-factor authentication (if properly implemented)
  ✗ Submit requests that require reading a secret from the page first
    (this is the basis of CSRF token defense)

NOTE:
  CSRF on GET requests is also possible but less common
  (GET actions should be idempotent — don't modify state on GET)
```

---

## 3. CSRF Attack Vectors

### Form-Based Attack

```html
<!-- Attacker's page: auto-submitting form -->
<form id="csrf" action="https://victim.com/settings/email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com" />
</form>
<script>
  document.getElementById("csrf").submit();
</script>
```

### Image Tag Attack (GET Requests)

```html
<!-- Exploits state-changing GET endpoints (bad practice by servers) -->
<img src="https://victim.com/admin/delete-user?id=42" />
<!-- Browser loads "image" → sends GET + session cookie → deletes user -->
```

### JavaScript Fetch Attack

```javascript
// If SameSite=None is set, or the cookie has no SameSite (legacy)
fetch("https://victim.com/api/transfer", {
  method: "POST",
  credentials: "include", // sends cookies
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: "to=attacker&amount=10000",
});
// Note: application/json with credentials: 'include' is a cross-origin request
// → triggers CORS preflight → server can reject it
// But: form-encoded content-type is "simple" → no preflight!
```

### Link Attack

```html
<!-- Disguised as a legitimate link -->
<a href="https://victim.com/delete-account?confirm=true">Free Prize!</a>
<!-- Clicking sends GET + cookies → deletes account -->
```

---

## 4. Defense 1 — SameSite Cookie Attribute

SameSite is the modern, most effective CSRF defense. It controls when cookies are sent with cross-site requests.

### SameSite Values

```http
# Strict: cookies only sent for same-site requests
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly

# With SameSite=Strict:
# User visits attacker.com → clicks link to victim.com:
#   → first request to victim.com: NO session cookie sent
#   → user appears logged out
# User navigating from victim.com's own links: session cookie sent normally
#
# CSRF protection: complete
# UX impact: navigating from external sites (email, other websites) = appears logged out
# Use for: banking, admin panels (security > convenience)

# Lax: sent on top-level navigations only (GET + navigation)
Set-Cookie: session=abc123; SameSite=Lax; Secure; HttpOnly

# With SameSite=Lax:
# GET link from attacker.com → victim.com: session cookie SENT (top-level nav)
# POST form from attacker.com to victim.com: session cookie NOT sent
# Fetch/XHR from attacker.com to victim.com: session cookie NOT sent
#
# CSRF protection: blocks most attacks (POST forms, XHR)
# Does not protect: GET state-changing endpoints
# Use for: most applications (DEFAULT in modern browsers since Chrome 80)

# None: sent in all contexts (old behavior)
Set-Cookie: session=abc123; SameSite=None; Secure; HttpOnly

# With SameSite=None:
# All cross-site requests send the cookie
# CSRF protection: NONE
# Use for: third-party cookies, SSO across domains, payment widgets
# Note: MUST include Secure when using SameSite=None
```

### The Lax Default

```
Since Chrome 80 (February 2020):
  Cookies without SameSite attribute: treated as SameSite=Lax
  This was a significant security improvement across the web

Modern application baseline:
  If you control the server: explicitly set SameSite=Strict or SameSite=Lax
  Don't rely on browser defaults (behavior may differ for legacy cookies)
```

### SameSite Limitations

```
SameSite=Lax does NOT protect against:
  1. GET state-changing endpoints (always use POST for mutations)
  2. Top-level navigation to the victim site with GET parameters
     <a href="https://bank.com/logout?csrf=1"> → works because top-level GET

SameSite=Lax does NOT protect CSRF for:
  Same-site !== same-origin!
  SameSite considers the registrable domain (example.com)
  If attacker controls app2.example.com: it's "same-site" as app.example.com
  → SameSite doesn't help for subdomain attacks
  Solution: use CSRF tokens for cross-subdomain scenarios

SameSite browser support:
  Chrome 80+: Lax by default
  Firefox 103+: Lax by default
  Safari: Strict by default for third-party
  IE11: no SameSite support (if you support IE, use CSRF tokens)
```

---

## 5. Defense 2 — CSRF Tokens (Synchronizer Token Pattern)

The classic CSRF defense: embed a secret token in every form, verify it server-side.

```
HOW IT WORKS:
  Server generates a unique random token per user session
  Token is embedded in every form (hidden field) and in pages via meta tag
  When form is submitted: token is included in POST body
  Server verifies: token in body matches token in session

  ATTACKER CANNOT:
  - Read the token from victim.com (SOP blocks cross-origin reads)
  - Guess the token (cryptographically random)
  - Forge the token without being able to read the page first
```

### Server-Side Token Generation

```javascript
// Node.js: generate CSRF tokens
import crypto from "crypto";

function generateCSRFToken() {
  return crypto.randomBytes(32).toString("hex"); // 64-char hex string
}

// Store in session on first request:
app.use((req, res, next) => {
  if (!req.session.csrfToken) {
    req.session.csrfToken = generateCSRFToken();
  }
  // Make token available in templates:
  res.locals.csrfToken = req.session.csrfToken;
  next();
});

// Inject into HTML templates:
// <meta name="csrf-token" content="{{ csrfToken }}">
// <input type="hidden" name="_csrf" value="{{ csrfToken }}">
```

### Server-Side Token Verification

```javascript
function csrfProtection(req, res, next) {
  // Only validate mutating methods
  if (["GET", "HEAD", "OPTIONS"].includes(req.method)) return next();

  const tokenFromBody = req.body._csrf;
  const tokenFromHeader = req.headers["x-csrf-token"];
  const token = tokenFromBody ?? tokenFromHeader;
  const expected = req.session.csrfToken;

  if (!token || !expected || !safeCompare(token, expected)) {
    return res.status(403).json({ error: "CSRF token validation failed" });
  }

  next();
}

// Timing-safe comparison (prevent timing attacks)
function safeCompare(a, b) {
  if (a.length !== b.length) return false;
  return crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b));
}

// Apply to all state-changing routes:
app.use("/api/", csrfProtection);
```

### Client-Side: Include Token in Requests

```javascript
// Option 1: Read from meta tag (set by server)
const csrfToken = document.querySelector('meta[name="csrf-token"]')?.content;

// Option 2: Read from cookie (Double Submit pattern — see Section 6)
const csrfToken = getCookie("XSRF-TOKEN");

// Include in all AJAX requests:
const apiClient = axios.create({
  headers: { "X-CSRF-Token": csrfToken },
});

// Or via interceptor:
axios.interceptors.request.use((config) => {
  const token = document.querySelector('meta[name="csrf-token"]')?.content;
  if (token) config.headers["X-CSRF-Token"] = token;
  return config;
});
```

### Per-Request Tokens (Stronger)

```javascript
// Per-request tokens: each request gets a unique token
// Prevents token reuse and sidechannel attacks

// Server generates token tied to session + timestamp
function generatePerRequestToken(sessionId) {
  const timestamp = Date.now();
  const hmac = crypto
    .createHmac("sha256", process.env.CSRF_SECRET)
    .update(`${sessionId}:${timestamp}`)
    .digest("hex");
  return `${timestamp}:${hmac}`;
}

function validatePerRequestToken(sessionId, token) {
  const [timestamp, hmac] = token.split(":");
  const age = Date.now() - Number(timestamp);

  if (age > 3600_000) return false; // expired (> 1 hour)

  const expected = crypto
    .createHmac("sha256", process.env.CSRF_SECRET)
    .update(`${sessionId}:${timestamp}`)
    .digest("hex");

  return safeCompare(hmac, expected);
}
```

---

## 6. Defense 3 — Double Submit Cookie

A stateless CSRF defense that doesn't require server-side session storage:

```
HOW IT WORKS:
  1. Server sets a random token as a cookie (non-HttpOnly)
     Set-Cookie: XSRF-TOKEN=random-secret; SameSite=Lax; Secure
     (Not HttpOnly: JavaScript must be able to read it)

  2. Client reads the cookie value with JavaScript
     const token = getCookie('XSRF-TOKEN');

  3. Client includes the value in every request header:
     X-XSRF-TOKEN: random-secret

  4. Server compares: cookie value === header value

WHY THIS WORKS:
  Attacker's page cannot read victim.com's cookies (SOP)
  So attacker cannot forge the X-XSRF-TOKEN header value
  The server verifies: "someone who could read the XSRF-TOKEN cookie sent this"
  Only same-origin JavaScript can read the cookie → must be legitimate request
```

```javascript
// Server: set XSRF-TOKEN cookie
app.use((req, res, next) => {
  if (!req.cookies["XSRF-TOKEN"]) {
    const token = crypto.randomBytes(32).toString("hex");
    res.cookie("XSRF-TOKEN", token, {
      secure: true,
      sameSite: "Lax",
      httpOnly: false, // JavaScript must read this
      maxAge: 3600_000,
    });
  }
  next();
});

// Server: validate double submit
function csrfDoubleSubmit(req, res, next) {
  if (["GET", "HEAD", "OPTIONS"].includes(req.method)) return next();

  const fromCookie = req.cookies["XSRF-TOKEN"];
  const fromHeader = req.headers["x-xsrf-token"];

  if (!fromCookie || !fromHeader || !safeCompare(fromCookie, fromHeader)) {
    return res.status(403).json({ error: "CSRF validation failed" });
  }

  next();
}

// Client: axios automatically reads XSRF-TOKEN cookie and sends as header
// (this is built into axios by default!)
axios.defaults.xsrfCookieName = "XSRF-TOKEN";
axios.defaults.xsrfHeaderName = "X-XSRF-TOKEN";
// No manual configuration needed with axios

// Fetch: must do it manually
function getCsrfToken() {
  return document.cookie
    .split("; ")
    .find((row) => row.startsWith("XSRF-TOKEN="))
    ?.split("=")[1];
}

async function csrfFetch(url, options = {}) {
  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      "X-XSRF-TOKEN": getCsrfToken() ?? "",
    },
  });
}
```

---

## 7. Defense 4 — Origin / Referer Header Validation

Check the Origin header (for CORS requests) or Referer header to verify request source:

```javascript
function validateOrigin(req, res, next) {
  if (["GET", "HEAD", "OPTIONS"].includes(req.method)) return next();

  const allowedOrigins = new Set([
    "https://app.example.com",
    "https://admin.example.com",
  ]);

  const origin = req.headers.origin;
  const referer = req.headers.referer;

  // Check Origin header (sent with most cross-origin requests)
  if (origin) {
    if (!allowedOrigins.has(origin)) {
      return res.status(403).json({ error: "Invalid origin" });
    }
    return next();
  }

  // Fall back to Referer header (may not be present, may be stripped)
  if (referer) {
    try {
      const refererOrigin = new URL(referer).origin;
      if (!allowedOrigins.has(refererOrigin)) {
        return res.status(403).json({ error: "Invalid referer" });
      }
      return next();
    } catch {
      return res.status(403).json({ error: "Invalid referer format" });
    }
  }

  // No Origin or Referer: reject (conservative approach)
  // Note: some browsers/proxies strip these headers
  // Consider allowing null Origin for same-origin requests from older setups
  return res.status(403).json({ error: "Missing origin headers" });
}
```

### Limitations of Origin/Referer Validation

```
Origin header:
  ✓ Sent for cross-origin requests
  ✗ Not always sent for same-origin requests
  ✗ Some privacy browsers/proxies strip it
  ✗ Null Origin can be spoofed in some scenarios

Referer header:
  ✗ Not sent on HTTPS → HTTP (downgrade)
  ✗ Stripped by Referrer-Policy: no-referrer
  ✗ Not reliable as sole defense

Recommendation: use as supplementary check, not primary defense
Primary: SameSite cookies + CSRF tokens
```

---

## 8. Defense 5 — Custom Request Headers

Any request with a custom header triggers CORS preflight for cross-origin requests. The preflight can be rejected at the CORS level.

```javascript
// All AJAX requests include a custom header
fetch("/api/transfer", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-Requested-With": "XMLHttpRequest", // custom header
  },
  body: JSON.stringify({ to: "alice", amount: 100 }),
});

// Cross-origin form submission CANNOT add custom headers
// → Simple form POST: no custom headers → no CORS preflight
// → XHR/fetch with custom header: CORS preflight → server rejects if origin not allowed

// Server: reject if custom header missing for API routes
function requireAjaxHeader(req, res, next) {
  const header = req.headers["x-requested-with"];
  if (header !== "XMLHttpRequest") {
    return res
      .status(403)
      .json({ error: "Direct form submission not allowed" });
  }
  next();
}

// NOTE: This defense works only if:
// 1. Your API accepts ONLY JSON (not form-encoded)
// 2. CORS is configured to reject unrecognized custom headers
// 3. Attacker cannot use CORS to add headers (blocked by SOP)
```

---

## 9. CSRF and Modern SPAs

Modern Single-Page Applications using JWT (not cookies) are largely immune to CSRF:

```
JWT IN AUTHORIZATION HEADER (CSRF immune):
  Client stores JWT in memory or localStorage
  Every request: manually includes header
  Authorization: Bearer eyJhbGciOiJ...

  CSRF attack: attacker's page cannot add Authorization header
  → Cross-origin requests without custom headers don't include the JWT
  → Server requires JWT → request rejected

  WHY: Custom headers trigger CORS preflight
       CORS rejects cross-origin requests without server permission
       → No Authorization header can be forged cross-site

JWT IN HTTPONLY COOKIE (CSRF vulnerable again!):
  Some apps store JWT in HttpOnly cookie for security
  But: HttpOnly cookies ARE sent automatically → CSRF is back
  Solution: SameSite cookie + CSRF token even for JWT apps
```

### SPA CSRF Defense Matrix

```
API AUTHENTICATION:
  JWT in Authorization header:    CSRF immune (can't forge custom headers)
  JWT in memory:                  CSRF immune (not sent automatically)
  JWT in cookie (HttpOnly):       CSRF vulnerable (use SameSite + CSRF token)
  Session cookie:                 CSRF vulnerable (use SameSite + CSRF token)

FETCH API vs FORM SUBMISSIONS:
  Fetch with credentials: 'omit': CSRF immune (cookies not sent)
  Fetch with credentials: 'include': CSRF vulnerable if cookie-based auth
  HTML form POST:          CSRF vulnerable (no CORS, cookies sent automatically)
```

---

## 10. CSRF vs XSS

These are frequently confused but fundamentally different:

```
                    XSS                         CSRF
What it exploits:  User trusts site             Site trusts user (browser)
Needs to read:     Yes (steals data)            No (just sends requests)
Requires user:     Visits malicious site        Visits malicious site
                   (or views stored content)
Attack vector:     Injected JavaScript          Forced requests (form, img, XHR)
Targets:           Client (browser state)       Server (stateful operations)
Can steal data:    Yes                          No (can't read cross-origin response)
Can take actions:  Yes                          Yes
Mitigated by:      Output encoding, CSP         SameSite cookies, CSRF tokens
HttpOnly helps:    Prevents cookie theft        Doesn't prevent (request still sent)

KEY RELATIONSHIP:
  XSS can bypass CSRF protection!
  If attacker can inject JavaScript into victim.com:
    - JavaScript can read the CSRF token from the page
    - Use the token to forge authenticated requests
  Solution: prevent XSS first, then CSRF protection works correctly
```

---

## 11. Logout CSRF

An often-overlooked variant: forcing a user to log out:

```html
<!-- Attacker page: force logout -->
<img src="https://victim.com/logout" />
<!-- If /logout is a GET endpoint: sends session cookie → logs user out
     Denial of service, or trick into logging in to attacker-controlled session -->

<!-- Defense: logout must be POST, not GET -->
<!-- And: require CSRF token on logout endpoint too -->
```

```javascript
// ✅ Logout via POST with CSRF token
app.post("/logout", csrfProtection, (req, res) => {
  req.session.destroy();
  res.clearCookie("session");
  res.redirect("/login");
});

// ❌ Logout via GET (CSRF vulnerable)
app.get("/logout", (req, res) => {
  req.session.destroy(); // any img tag can log user out!
  res.redirect("/login");
});
```

---

## 12. Good Practices

### ✅ Use SameSite=Strict for high-security actions

```javascript
// ✅ Strict for sessions that control sensitive actions
res.cookie("session", token, {
  httpOnly: true,
  secure: true,
  sameSite: "Strict", // Maximum protection
  maxAge: 3600_000,
});
```

### ✅ Combine multiple defenses

```javascript
// ✅ Defense in depth: multiple independent mechanisms
// 1. SameSite cookies (primary)
// 2. CSRF token (secondary — handles edge cases)
// 3. Origin validation (tertiary — additional check)
// Each defense independently prevents CSRF
// All three together make CSRF essentially impossible
```

### ✅ Use state-changing verbs correctly

```javascript
// ✅ POST/PUT/PATCH/DELETE for state changes
app.delete("/api/users/:id", auth, csrfProtection, deleteUser);
app.post("/api/orders", auth, csrfProtection, createOrder);

// ❌ GET for state changes (any <img> tag can trigger this)
app.get("/api/users/:id/delete", deleteUser); // never do this
```

---

## 13. Bad Practices

### ❌ Checking only Referer (too easy to bypass)

```javascript
// ❌ Referer as sole CSRF defense — unreliable
app.use((req, res, next) => {
  const referer = req.headers.referer;
  if (referer && referer.startsWith("https://example.com")) {
    return next();
  }
  return res.status(403).end();
});
// Problems:
// - Referer stripped by privacy tools, Referrer-Policy headers
// - example.com.evil.com starts with 'https://example.com' (partial match bug)
// - Legitimate requests may not have Referer
```

### ❌ Using CSRF tokens but transmitting via GET

```javascript
// ❌ CSRF token in URL is logged and leaked via Referer
app.get("/api/action?csrf_token=abc123&action=delete", handler);
// Token in URL: appears in server logs, browser history, Referer header
// Attacker can steal token from Referer header on same page

// ✅ CSRF token in POST body or request header only
app.post("/api/action", csrfProtection, handler);
// Token in POST body: not logged, not leaked
```

### ❌ Same CSRF token for every user (shared secret)

```javascript
// ❌ Same token for all users or across sessions
const CSRF_TOKEN = "static-token"; // never do this

// ✅ Unique random token per user session
req.session.csrfToken = crypto.randomBytes(32).toString("hex");
```

---

## 14. Common Mistakes

### Mistake 1 — SameSite=None without HTTPS

```javascript
// ❌ SameSite=None requires Secure
res.cookie("widget-auth", token, {
  sameSite: "None", // meant for cross-site widgets
  // missing: secure: true
});
// Browser silently ignores SameSite=None without Secure
// → Cookie treated as SameSite=Lax (may break third-party widget)

// ✅ Always include Secure with SameSite=None
res.cookie("widget-auth", token, {
  sameSite: "None",
  secure: true,
  httpOnly: true,
});
```

### Mistake 2 — Not regenerating CSRF token after login

```javascript
// ❌ Session fixation via CSRF token
// Pre-authentication: user gets CSRF token X
// After login: token X still valid
// Attack: attacker injects CSRF token X into victim's pre-auth session
//         victim logs in → token X valid for authenticated requests

// ✅ Regenerate CSRF token after authentication
app.post("/login", async (req, res) => {
  const user = await authenticate(req.body);
  if (user) {
    req.session.regenerate(() => {
      // new session ID
      req.session.userId = user.id;
      req.session.csrfToken = crypto.randomBytes(32).toString("hex"); // new CSRF token
      res.redirect("/dashboard");
    });
  }
});
```

### Mistake 3 — Forgetting CSRF protection on JSON endpoints

```javascript
// ❌ Assuming JSON APIs are automatically CSRF-safe
app.post("/api/transfer", authenticate, handler);
// JSON APIs can still be CSRF targets!
// If Content-Type: application/x-www-form-urlencoded:
//   attacker's form can submit to JSON endpoint
// Server parses as JSON → fails
// But if server accepts both: vulnerable!

// ✅ For APIs that only accept JSON:
app.post(
  "/api/transfer",
  (req, res, next) => {
    if (!req.is("application/json")) {
      return res.status(415).json({ error: "Only JSON accepted" });
    }
    next();
  },
  authenticate,
  handler,
);
// JSON with custom header triggers CORS → protects API
// Form submit as JSON → rejected by content-type check
```

---

## 15. Interview-Level Explanation

> **"What is CSRF? How do you protect against it in a modern web application?"**

**Strong answer:**

> "CSRF — Cross-Site Request Forgery — exploits the browser's automatic cookie sending. When a user is logged into bank.com, their session cookie is stored in the browser. If they then visit attacker.com, attacker.com can include a form or fetch request targeting bank.com. The browser automatically sends the bank.com session cookie with that request — the server sees a valid authenticated request and processes it. The attacker never reads the response; they just need the request to happen.
>
> The modern primary defense is the SameSite cookie attribute. With `SameSite=Strict`, the session cookie is only sent when the request originates from the same site — cross-site requests include no cookie. With `SameSite=Lax` (the default in Chrome 80+), cookies are sent on top-level GET navigations but not on cross-site POST forms, XHR, or fetch. For most applications, Lax provides good protection; Strict is appropriate for high-security contexts like banking.
>
> SameSite has limits: it doesn't protect GET endpoints that modify state (always use POST for mutations), and 'same-site' is broader than 'same-origin' — if an attacker controls a subdomain of your site, they're considered 'same-site' and cookies are still sent. For these cases, CSRF tokens provide a second defense: the server embeds a random secret in the page, the client includes it in every mutating request, and the server verifies the match. An attacker on evil.com can't read the victim's page to steal the token (Same-Origin Policy blocks cross-origin reads), so they can't forge a request with a valid token.
>
> For modern SPAs using JWT in the Authorization header: CSRF isn't a concern because custom headers can't be added to cross-site requests without CORS approval. But if JWT is stored in a cookie, CSRF protection is required again.
>
> The relationship with XSS: XSS can defeat CSRF protection because injected JavaScript can read the CSRF token from the page. You need to prevent XSS first; otherwise CSRF tokens are undermined."

---

## 16. Exercises

### Exercise 1 — Identify CSRF vulnerabilities

```javascript
// Which of these endpoints are CSRF vulnerable? How would you fix each?

// A: State-changing GET
app.get("/settings/notifications", auth, (req, res) => {
  const { enabled } = req.query;
  db.users.update(req.user.id, { notifications: enabled === "true" });
  res.redirect("/settings");
});

// B: POST with JSON, no CSRF protection
app.post("/api/profile", auth, (req, res) => {
  const { name, email } = req.body;
  db.users.update(req.user.id, { name, email });
  res.json({ success: true });
});

// C: POST form with CSRF token check
app.post("/delete-account", auth, csrfProtection, (req, res) => {
  db.users.delete(req.user.id);
  res.redirect("/goodbye");
});

// D: DELETE with SameSite=Strict session cookie, no CSRF token
app.delete("/api/posts/:id", auth, (req, res) => {
  db.posts.delete(req.params.id);
  res.status(204).end();
});
```

<details>
<summary>Answer</summary>

```
A: CSRF VULNERABLE (state-changing GET)
   Problem: any <img> or <link> can trigger state change
   Attack: <img src="https://victim.com/settings/notifications?enabled=false">
   Fix: Change to POST endpoint + CSRF protection
   app.post('/settings/notifications', auth, csrfProtection, handler);

B: POTENTIALLY VULNERABLE (depends on Content-Type handling)
   If server accepts application/x-www-form-urlencoded too: CSRF possible
   Cross-origin JSON fetch requires CORS preflight → may be protected
   BUT: if server allows both JSON and form-encoded, attacker can use form
   Fix: Explicitly require JSON content-type + add CSRF token
   if (!req.is('application/json')) return res.status(415).end();
   // OR: add CSRF token validation

C: PROTECTED (POST + CSRF token = correct defense)
   ✓ POST (not GET)
   ✓ csrfProtection middleware validates token
   No vulnerability here

D: DEPENDS ON COOKIE CONFIGURATION
   With SameSite=Strict: PROTECTED (DELETE won't carry cross-site cookies)
   With SameSite=Lax: DEPENDS (Lax doesn't protect non-navigation requests,
                      but DELETE is a non-navigation → cookie NOT sent → protected)
   With no SameSite: VULNERABLE (old browser or legacy cookie)

   Best practice: add CSRF token even with SameSite for defense in depth
   Especially because: 'same-site' ≠ 'same-origin' (subdomain issue)
```

</details>

---

## 🔗 Related Topics

- [`security/01-xss.md`](./01-xss.md) — XSS can bypass CSRF protection
- [`networking/04-cors-and-security.md`](../networking/04-cors-and-security.md) — CORS as related defense
- [`security/03-headers.md`](./03-headers.md) — Security headers including cookie security
- [`networking/02-fetch-and-xhr.md`](../networking/02-fetch-and-xhr.md) — Fetch credentials and CORS

---

<div align="center">

**Next:** [`security/03-headers.md`](./03-headers.md) →

</div>
