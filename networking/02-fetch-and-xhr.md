# 02 — Fetch and XHR

> **"Fetch is not just 'the modern XHR.' It's a fundamentally different mental model: Promises instead of callbacks, streams instead of buffered responses, and composable by design. Understanding the difference between Fetch's two-stage fulfillment, its streaming model, and its CORS semantics is the difference between using it and truly understanding it."**

Fetch and XMLHttpRequest are the two browser APIs for making HTTP requests. XHR arrived in 1999 and powered the original AJAX era. Fetch arrived in 2015 as a Promise-based, stream-oriented, more composable replacement. This document covers both APIs in depth: Fetch's two-phase design, streaming, AbortController, request/response composition, CORS, credentials, and the patterns that make fetch-based code reliable and performant in production.

---

## 📚 Table of Contents

1. [Fetch vs XHR — The Mental Model Shift](#1-fetch-vs-xhr--the-mental-model-shift)
2. [The Fetch Two-Phase Design](#2-the-fetch-two-phase-design)
3. [Request Construction](#3-request-construction)
4. [Response Handling](#4-response-handling)
5. [Streaming Responses](#5-streaming-responses)
6. [AbortController — Cancellation](#6-abortcontroller--cancellation)
7. [CORS and Credentials](#7-cors-and-credentials)
8. [XMLHttpRequest Deep Dive](#8-xmlhttprequest-deep-dive)
9. [Progress Tracking](#9-progress-tracking)
10. [Request Retries and Exponential Backoff](#10-request-retries-and-exponential-backoff)
11. [Building a Production Fetch Client](#11-building-a-production-fetch-client)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. Fetch vs XHR — The Mental Model Shift

### XHR: Event-Driven, Stateful

```javascript
// XHR: event listeners, state machine, imperative
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/users");
xhr.setRequestHeader("Authorization", `Bearer ${token}`);

xhr.onload = () => {
  if (xhr.status >= 200 && xhr.status < 300) {
    const data = JSON.parse(xhr.responseText);
    processData(data);
  } else {
    handleError(xhr.status);
  }
};

xhr.onerror = () => handleNetworkError();
xhr.onabort = () => handleAbort();
xhr.onprogress = (e) => updateProgress(e.loaded, e.total);

xhr.send();

// State machine:
// UNSENT → OPENED → HEADERS_RECEIVED → LOADING → DONE
// You can't compose, cancel cleanly, or stream with XHR
```

### Fetch: Promise-Based, Composable

```javascript
// Fetch: Promises, composable Request/Response objects, streams
const response = await fetch("/api/users", {
  headers: { Authorization: `Bearer ${token}` },
});

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}

const data = await response.json();
processData(data);
```

### Critical Difference — Fetch Doesn't Reject on HTTP Errors

```javascript
// ❌ Common mistake: treating HTTP errors as Promise rejections
try {
  const response = await fetch('/api/users');
  const data = await response.json(); // succeeds even on 404/500!
} catch (err) {
  // Only catches: network failure, CORS error, DNS failure
  // Does NOT catch: 404, 500, 401 — those are fulfilled Promises!
}

// ✅ Correct: check response.ok
async function safeFetch(url, options) {
  const response = await fetch(url, options);
  if (!response.ok) {
    // response.ok = true if status is 200-299
    const errorText = await response.text().catch(() => response.statusText);
    throw new HttpError(response.status, errorText, url);
  }
  return response;
}

class HttpError extends Error {
  constructor(
    public status: number,
    message: string,
    public url: string
  ) {
    super(`${status}: ${message} (${url})`);
    this.name = 'HttpError';
  }
}
```

---

## 2. The Fetch Two-Phase Design

Fetch has two distinct phases, each returning a separate Promise:

```
PHASE 1: Response received (headers arrived)
  const response = await fetch(url);
  At this point: headers are available
  Body: NOT yet consumed (still streaming from network)

PHASE 2: Body consumed
  const data = await response.json();    // or
  const text = await response.text();    // or
  const blob = await response.blob();    // or
  const buffer = await response.arrayBuffer();
  At this point: body fully received and parsed
```

### Why Two Phases?

```javascript
// Phase 1 gives you: status, headers, URL, redirected, type
// Before downloading the full body

// Use case: check content type before consuming body
const response = await fetch("/api/data");

// Phase 1: headers available — can decide what to do with the body
const contentType = response.headers.get("Content-Type");
const contentLength = response.headers.get("Content-Length");
console.log(`Incoming: ${contentType}, ${contentLength} bytes`);

if (contentType.includes("application/json")) {
  const data = await response.json(); // Phase 2: download + parse JSON
} else if (contentType.includes("text/")) {
  const text = await response.text(); // Phase 2: download as text
} else {
  const blob = await response.blob(); // Phase 2: download as Blob
}

// Use case: check response code before parsing body (save time on errors)
const response = await fetch("/api/large-dataset");
if (!response.ok) {
  // Don't bother downloading the error body if we don't need it
  throw new Error(`Failed: ${response.status}`);
}
// Only proceed to Phase 2 if the response was successful
const data = await response.json();
```

### Body Can Only Be Consumed Once

```javascript
// ❌ Body consumed twice: second read returns null/error
const response = await fetch("/api/data");
const json1 = await response.json(); // consumes body ✓
const json2 = await response.json(); // TypeError: body already consumed! ✗

// ✅ Clone the response before consuming if you need it twice
const response = await fetch("/api/data");
const clone = response.clone(); // clone before consuming

const json = await response.json(); // consume original
const text = await clone.text(); // consume clone separately

// Common use case: log raw response AND parse JSON
const response = await fetch("/api/data");
const cloneForLogging = response.clone();
const data = await response.json();
logger.debug("Raw response:", await cloneForLogging.text());
```

---

## 3. Request Construction

### The Request Object

```javascript
// Method 1: URL + init object (most common)
const response = await fetch("/api/users", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${token}`,
    "X-Request-ID": generateId(),
  },
  body: JSON.stringify(userData),
  mode: "cors", // 'cors' | 'no-cors' | 'same-origin'
  credentials: "include", // 'include' | 'same-origin' | 'omit'
  cache: "no-cache", // 'default' | 'no-store' | 'reload' | 'no-cache' | 'force-cache'
  redirect: "follow", // 'follow' | 'error' | 'manual'
  referrerPolicy: "no-referrer-when-downgrade",
  signal: controller.signal, // AbortController signal
  priority: "high", // 'high' | 'low' | 'auto'
});

// Method 2: Request object (composable, reusable)
const baseRequest = new Request("/api/users", {
  headers: { Authorization: `Bearer ${token}` },
});

// Create variants from base:
const postRequest = new Request(baseRequest, {
  method: "POST",
  body: JSON.stringify(newUser),
});
```

### Request Body Types

```javascript
// JSON (most common):
fetch("/api/create", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Alice", email: "alice@example.com" }),
});

// Form data (multipart or URL-encoded):
const formData = new FormData();
formData.append("name", "Alice");
formData.append("avatar", fileInput.files[0], "avatar.jpg");

fetch("/api/upload", {
  method: "POST",
  body: formData, // Content-Type set automatically (multipart/form-data with boundary)
});

// URL-encoded form data:
const params = new URLSearchParams({ name: "Alice", role: "admin" });
fetch("/api/form", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: params.toString(),
});

// Binary data (ArrayBuffer):
const buffer = await readFileAsBuffer(file);
fetch("/api/binary", {
  method: "POST",
  headers: { "Content-Type": "application/octet-stream" },
  body: buffer,
});

// Blob:
const blob = new Blob([JSON.stringify(data)], { type: "application/json" });
fetch("/api/data", { method: "POST", body: blob });

// ReadableStream (streaming request body):
const stream = new ReadableStream({
  start(controller) {
    // Push data chunks
    controller.enqueue(new TextEncoder().encode("chunk 1"));
    controller.enqueue(new TextEncoder().encode("chunk 2"));
    controller.close();
  },
});
fetch("/api/stream", { method: "POST", body: stream, duplex: "half" });
```

---

## 4. Response Handling

### Response Properties

```javascript
const response = await fetch("/api/data");

// Status
response.status; // 200, 404, 500, etc.
response.ok; // true if 200-299
response.statusText; // "OK", "Not Found", "Internal Server Error"

// Headers (read-only, iterable)
response.headers.get("Content-Type");
response.headers.get("ETag");
response.headers.get("X-RateLimit-Remaining");
[...response.headers.entries()]; // all header key-value pairs

// URL and redirect
response.url; // final URL (after redirects)
response.redirected; // true if redirected from original URL
response.type; // 'basic' | 'cors' | 'opaque' | 'opaqueredirect'

// Body
response.body; // ReadableStream<Uint8Array>
response.bodyUsed; // true once body is consumed
```

### Response Body Methods

```javascript
// JSON (most common)
const data = await response.json();

// Text
const html = await response.text();

// Binary
const buffer = await response.arrayBuffer();

// Blob (for files, images)
const blob = await response.blob();
const url = URL.createObjectURL(blob);
img.src = url;

// FormData (if server responds with form data)
const formData = await response.formData();

// Body stream directly
const reader = response.body.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  processChunk(value); // Uint8Array
}
```

---

## 5. Streaming Responses

Streaming lets you process large responses before they fully download:

```javascript
// Progress tracking + streaming JSON
async function* streamLargeResponse(url) {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });

      // Parse complete lines (newline-delimited JSON)
      const lines = buffer.split("\n");
      buffer = lines.pop() ?? ""; // keep incomplete last line

      for (const line of lines) {
        if (line.trim()) {
          yield JSON.parse(line);
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
}

// Process items as they arrive
for await (const item of streamLargeResponse("/api/stream")) {
  displayItem(item); // render before full response arrives
}
```

### Server-Sent Events via Fetch

```javascript
// Server sends: "data: {...}\n\n"
async function subscribeToSSE(url, onEvent) {
  const response = await fetch(url, {
    headers: { Accept: "text/event-stream" },
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });

    // Parse SSE format: "data: {...}\n\n"
    const messages = buffer.split("\n\n");
    buffer = messages.pop() ?? "";

    for (const message of messages) {
      const dataLine = message.split("\n").find((l) => l.startsWith("data: "));
      if (dataLine) {
        const data = JSON.parse(dataLine.slice(6));
        onEvent(data);
      }
    }
  }
}

// Usage:
subscribeToSSE("/api/events", (event) => {
  console.log("Event received:", event);
});
```

### Response Streaming with Progress

```javascript
async function fetchWithProgress(url, onProgress) {
  const response = await fetch(url);
  const contentLength = Number(response.headers.get("Content-Length"));
  const reader = response.body.getReader();
  const chunks = [];
  let received = 0;

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    chunks.push(value);
    received += value.length;

    if (contentLength) {
      onProgress(received / contentLength); // 0 to 1
    } else {
      onProgress(received); // bytes received (no total)
    }
  }

  // Concatenate all chunks into one Uint8Array
  const total = new Uint8Array(received);
  let offset = 0;
  for (const chunk of chunks) {
    total.set(chunk, offset);
    offset += chunk.length;
  }

  return new Response(total, response);
}
```

---

## 6. AbortController — Cancellation

```javascript
// Basic abort
const controller = new AbortController();
const signal = controller.signal;

// Start request
const fetchPromise = fetch("/api/data", { signal });

// Cancel after 5 seconds
const timeout = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetchPromise;
  clearTimeout(timeout);
  return await response.json();
} catch (err) {
  if (err.name === "AbortError") {
    console.log("Request aborted");
    return null;
  }
  throw err; // non-abort errors
}
```

### Timeout Pattern

```javascript
// Fetch with timeout (built-in in Node.js 18+, polyfill for browsers)
async function fetchWithTimeout(url, options = {}, timeoutMs = 5000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort("timeout"), timeoutMs);

  try {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal,
    });
    clearTimeout(timeoutId);
    return response;
  } catch (err) {
    clearTimeout(timeoutId);
    if (err.name === "AbortError") {
      throw new Error(`Request to ${url} timed out after ${timeoutMs}ms`);
    }
    throw err;
  }
}

// AbortSignal.timeout (Chrome 103+, Firefox 100+):
const response = await fetch("/api/data", {
  signal: AbortSignal.timeout(5000), // auto-aborts after 5 seconds
});
```

### React: Cancel Fetch on Unmount

```typescript
// Cancel in-flight requests when component unmounts
function useDataFetch(url: string) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    async function fetchData() {
      setLoading(true);
      try {
        const response = await fetch(url, { signal: controller.signal });
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const json = await response.json();
        setData(json);
      } catch (err) {
        if (err.name !== "AbortError") {
          setError(err as Error);
        }
        // AbortError: component unmounted — silently ignore
      } finally {
        setLoading(false);
      }
    }

    fetchData();

    return () => controller.abort(); // cleanup: abort on unmount
  }, [url]);

  return { data, loading, error };
}
```

### Combining Multiple Abort Signals

```javascript
// AbortSignal.any([...]) — abort when ANY signal fires
// (Chrome 116+, Firefox 120+)
const userAbort = new AbortController();
const timeoutSig = AbortSignal.timeout(10_000);

const response = await fetch("/api/large-data", {
  signal: AbortSignal.any([userAbort.signal, timeoutSig]),
  // Aborts if user cancels OR 10 seconds pass
});

// Button: let user cancel
cancelButton.onclick = () => userAbort.abort();
```

---

## 7. CORS and Credentials

### CORS Modes

```javascript
// mode: 'cors' (default)
// Can make cross-origin requests
// Server must respond with correct CORS headers
// Browser blocks response if CORS headers missing

fetch("https://api.other-domain.com/data", {
  mode: "cors", // explicit (same as default)
});

// mode: 'same-origin'
// Only allows same-origin requests
// Throws TypeError for cross-origin requests
// Useful for: assert no cross-origin requests in sensitive code

fetch("/api/users", { mode: "same-origin" });

// mode: 'no-cors'
// Can make cross-origin requests to any server
// BUT: response is "opaque" — you can't read status, headers, or body
// Only useful for: fire-and-forget analytics pings, caching opaque images

fetch("https://analytics.example.com/ping", {
  method: "POST",
  mode: "no-cors",
  body: JSON.stringify(event),
  // Status: always 0, body: always null
});
```

### Credentials (Cookies, Auth)

```javascript
// credentials: 'same-origin' (default)
// Sends cookies and auth headers ONLY for same-origin requests
// Cross-origin: no cookies sent

// credentials: 'include'
// Sends cookies and auth headers for ALL requests (cross-origin too)
// Requires: server sets Access-Control-Allow-Credentials: true
// AND Access-Control-Allow-Origin must be a specific origin (not *)

fetch("https://api.example.com/user", {
  credentials: "include", // send session cookie cross-origin
});
// Server must respond:
// Access-Control-Allow-Origin: https://app.example.com (specific, not *)
// Access-Control-Allow-Credentials: true

// credentials: 'omit'
// Never send cookies or auth headers
// Use for: public APIs, analytics, third-party calls
// Also: mode: 'no-cors' implies credentials: 'same-origin'

fetch("/api/public", { credentials: "omit" });
```

### Preflight Requests

```javascript
// "Simple" requests: no preflight
// - Methods: GET, HEAD, POST
// - Content-Type: text/plain, application/x-www-form-urlencoded, multipart/form-data
// - No custom headers

// "Complex" requests: browser sends OPTIONS preflight first
// Triggers preflight:
fetch("/api/data", {
  method: "PUT", // non-simple method
  headers: { "X-Custom-Header": "value" }, // custom header
  body: JSON.stringify(data),
  // Content-Type: application/json → triggers preflight
});

// Preflight exchange:
// → OPTIONS /api/data
//   Origin: https://app.example.com
//   Access-Control-Request-Method: PUT
//   Access-Control-Request-Headers: Content-Type, X-Custom-Header
//
// ← 200 OK
//   Access-Control-Allow-Origin: https://app.example.com
//   Access-Control-Allow-Methods: PUT, PATCH, DELETE
//   Access-Control-Allow-Headers: Content-Type, X-Custom-Header
//   Access-Control-Max-Age: 86400   ← cache preflight for 24 hours

// Reducing preflights:
// 1. Use GET/POST for reads (no preflight)
// 2. Set Access-Control-Max-Age on server to cache preflight results
// 3. Consolidate requests to reduce total preflight count
```

---

## 8. XMLHttpRequest Deep Dive

XHR is necessary for upload progress events (Fetch upload progress is limited) and older browser support:

```javascript
// Complete XHR pattern with all event handlers
function xhrRequest(url, options = {}) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();

    xhr.open(options.method ?? "GET", url, true); // true = async

    // Set headers
    Object.entries(options.headers ?? {}).forEach(([key, value]) => {
      xhr.setRequestHeader(key, value);
    });

    // Configure
    xhr.timeout = options.timeout ?? 0;
    xhr.withCredentials = options.credentials === "include";
    xhr.responseType = options.responseType ?? ""; // '' | 'arraybuffer' | 'blob' | 'json' | 'text'

    // Event handlers
    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve({
          status: xhr.status,
          headers: parseHeaders(xhr.getAllResponseHeaders()),
          data: xhr.response,
        });
      } else {
        reject(new Error(`HTTP ${xhr.status}: ${xhr.statusText}`));
      }
    };

    xhr.onerror = () => reject(new Error("Network error"));
    xhr.ontimeout = () => reject(new Error(`Timeout after ${xhr.timeout}ms`));
    xhr.onabort = () => reject(new Error("Request aborted"));

    // Progress (download)
    xhr.onprogress = (e) => {
      if (e.lengthComputable && options.onDownloadProgress) {
        options.onDownloadProgress(e.loaded / e.total);
      }
    };

    // Upload progress (XHR's killer feature vs Fetch)
    if (xhr.upload && options.onUploadProgress) {
      xhr.upload.onprogress = (e) => {
        if (e.lengthComputable) {
          options.onUploadProgress(e.loaded / e.total);
        }
      };
    }

    xhr.send(options.body ?? null);

    // Return cancellation function
    options.signal?.addEventListener("abort", () => xhr.abort());
  });
}
```

### Upload Progress with XHR

```javascript
// ✅ XHR for upload progress (Fetch doesn't support upload progress well)
function uploadFile(file, url, onProgress) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    const formData = new FormData();
    formData.append("file", file);

    xhr.open("POST", url, true);

    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        const percent = ((e.loaded / e.total) * 100).toFixed(0);
        onProgress(Number(percent));
      }
    };

    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve(JSON.parse(xhr.responseText));
      } else {
        reject(new Error(`Upload failed: ${xhr.status}`));
      }
    };

    xhr.onerror = () => reject(new Error("Upload error"));
    xhr.send(formData);
  });
}

// Usage
await uploadFile(
  videoFile,
  "/api/videos/upload",
  (percent) => (progressBar.style.width = `${percent}%`),
);
```

---

## 9. Progress Tracking

### Download Progress via Fetch Streams

```javascript
async function fetchWithDownloadProgress(url, onProgress) {
  const response = await fetch(url);
  const total = Number(response.headers.get("Content-Length") ?? 0);
  const reader = response.body.getReader();
  const chunks = [];
  let received = 0;

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    chunks.push(value);
    received += value.length;
    onProgress(total ? received / total : received);
  }

  const data = new Uint8Array(received);
  let offset = 0;
  for (const chunk of chunks) {
    data.set(chunk, offset);
    offset += chunk.length;
  }

  return data.buffer;
}

// Usage with visual progress:
const buffer = await fetchWithDownloadProgress(
  "/downloads/large-file.zip",
  (progress) => {
    if (progress <= 1) {
      progressBar.style.width = `${(progress * 100).toFixed(0)}%`;
    } else {
      bytesLabel.textContent = `${(progress / 1024).toFixed(1)} KB downloaded`;
    }
  },
);
```

---

## 10. Request Retries and Exponential Backoff

```typescript
interface RetryOptions {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  retryOn: (response: Response | null, error: Error | null) => boolean;
  onRetry?: (
    attempt: number,
    error: Error | null,
    response: Response | null,
  ) => void;
  signal?: AbortSignal;
}

async function fetchWithRetry(
  url: RequestInfo,
  options: RequestInit = {},
  retry: Partial<RetryOptions> = {},
): Promise<Response> {
  const {
    maxAttempts = 3,
    baseDelayMs = 1000,
    maxDelayMs = 30_000,
    retryOn = defaultRetryOn,
    onRetry,
    signal,
  } = retry;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    if (signal?.aborted) throw new DOMException("Aborted", "AbortError");

    let response: Response | null = null;
    let lastError: Error | null = null;

    try {
      response = await fetch(url, { ...options, signal });
    } catch (err) {
      lastError = err as Error;
      if (err.name === "AbortError") throw err; // never retry aborts
    }

    // Check if we should retry
    if (attempt < maxAttempts && retryOn(response, lastError)) {
      onRetry?.(attempt, lastError, response);

      // Exponential backoff with jitter
      const delay = Math.min(
        baseDelayMs * 2 ** (attempt - 1) + Math.random() * 1000,
        maxDelayMs,
      );

      await new Promise((resolve, reject) => {
        const timeoutId = setTimeout(resolve, delay);
        signal?.addEventListener("abort", () => {
          clearTimeout(timeoutId);
          reject(new DOMException("Aborted", "AbortError"));
        });
      });

      continue; // retry
    }

    // No more retries
    if (lastError) throw lastError;
    return response!;
  }

  throw new Error("Max retry attempts exceeded");
}

function defaultRetryOn(
  response: Response | null,
  error: Error | null,
): boolean {
  if (error) return true; // network errors: always retry
  if (!response) return false;
  // Retry on server errors and rate limiting
  return response.status === 429 || response.status >= 500;
}

// Usage:
const response = await fetchWithRetry(
  "/api/data",
  { headers: { Authorization: `Bearer ${token}` } },
  {
    maxAttempts: 3,
    baseDelayMs: 1000,
    onRetry: (attempt, error, response) => {
      console.warn(
        `Retry ${attempt}: ${error?.message ?? `HTTP ${response?.status}`}`,
      );
    },
  },
);
```

---

## 11. Building a Production Fetch Client

```typescript
// A typed, feature-complete fetch client for production use

interface FetchClientOptions {
  baseURL: string;
  timeout: number;
  headers: Record<string, string>;
  onRequest?: (config: RequestConfig) => RequestConfig | Promise<RequestConfig>;
  onResponse?: (response: Response) => Response | Promise<Response>;
  onError?: (error: FetchClientError) => void;
}

interface RequestConfig extends RequestInit {
  url: string;
  params?: Record<string, string | number | boolean | undefined>;
}

class FetchClient {
  #options: FetchClientOptions;

  constructor(options: Partial<FetchClientOptions> = {}) {
    this.#options = {
      baseURL: "",
      timeout: 10_000,
      headers: { "Content-Type": "application/json" },
      ...options,
    };
  }

  async request<T>(config: RequestConfig): Promise<T> {
    // Run request interceptor
    const finalConfig = this.#options.onRequest
      ? await this.#options.onRequest(config)
      : config;

    // Build URL
    const url = this.#buildURL(finalConfig.url, finalConfig.params);

    // Merge headers
    const headers = {
      ...this.#options.headers,
      ...Object.fromEntries(new Headers(finalConfig.headers).entries()),
    };

    // Create AbortController for timeout
    const controller = new AbortController();
    const timeoutId = setTimeout(
      () => controller.abort("timeout"),
      this.#options.timeout,
    );

    // Combine with provided signal
    const signal = finalConfig.signal
      ? AbortSignal.any([finalConfig.signal, controller.signal])
      : controller.signal;

    try {
      let response = await fetch(url, {
        ...finalConfig,
        headers,
        signal,
      });
      clearTimeout(timeoutId);

      // Run response interceptor
      if (this.#options.onResponse) {
        response = await this.#options.onResponse(response);
      }

      // Check HTTP status
      if (!response.ok) {
        let errorData: unknown;
        try {
          errorData = await response.clone().json();
        } catch {}
        throw new FetchClientError(
          response.status,
          response.statusText,
          url,
          errorData,
        );
      }

      // Parse response
      const contentType = response.headers.get("Content-Type") ?? "";
      if (!response.body || response.status === 204) return null as T;
      if (contentType.includes("application/json"))
        return response.json() as Promise<T>;
      if (contentType.includes("text/")) return response.text() as unknown as T;
      return response.blob() as unknown as T;
    } catch (err) {
      clearTimeout(timeoutId);
      if (err instanceof FetchClientError) {
        this.#options.onError?.(err);
        throw err;
      }
      const clientError = new FetchClientError(0, err.message, url);
      this.#options.onError?.(clientError);
      throw clientError;
    }
  }

  get<T>(url: string, params?: RequestConfig["params"], options?: RequestInit) {
    return this.request<T>({ url, method: "GET", params, ...options });
  }

  post<T>(url: string, body?: unknown, options?: RequestInit) {
    return this.request<T>({
      url,
      method: "POST",
      body: JSON.stringify(body),
      ...options,
    });
  }

  put<T>(url: string, body?: unknown, options?: RequestInit) {
    return this.request<T>({
      url,
      method: "PUT",
      body: JSON.stringify(body),
      ...options,
    });
  }

  patch<T>(url: string, body?: unknown, options?: RequestInit) {
    return this.request<T>({
      url,
      method: "PATCH",
      body: JSON.stringify(body),
      ...options,
    });
  }

  delete<T>(url: string, options?: RequestInit) {
    return this.request<T>({ url, method: "DELETE", ...options });
  }

  #buildURL(
    path: string,
    params?: Record<string, string | number | boolean | undefined>,
  ): string {
    const url = new URL(path, this.#options.baseURL || window.location.origin);
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined && value !== null) {
          url.searchParams.set(key, String(value));
        }
      });
    }
    return url.toString();
  }
}

class FetchClientError extends Error {
  constructor(
    public status: number,
    message: string,
    public url: string,
    public data?: unknown,
  ) {
    super(`[${status}] ${message} (${url})`);
    this.name = "FetchClientError";
  }
  get isNetworkError() {
    return this.status === 0;
  }
  get isUnauthorized() {
    return this.status === 401;
  }
  get isForbidden() {
    return this.status === 403;
  }
  get isNotFound() {
    return this.status === 404;
  }
  get isServerError() {
    return this.status >= 500;
  }
}

// Singleton client:
export const apiClient = new FetchClient({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10_000,
  headers: { "X-App-Version": APP_VERSION },
  onRequest: async (config) => ({
    ...config,
    headers: {
      ...config.headers,
      Authorization: `Bearer ${await getAuthToken()}`,
    },
  }),
  onError: (error) => {
    if (error.isUnauthorized) clearAuthAndRedirectToLogin();
    errorMonitor.capture(error);
  },
});
```

---

## 12. Good Practices

### ✅ Always check `response.ok` before reading body

```javascript
// ✅ Explicit HTTP error handling
const response = await fetch(url);
if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${await response.text()}`);
}
```

### ✅ Use AbortSignal.timeout for all requests

```javascript
// ✅ Built-in timeout without manual cleanup
const response = await fetch("/api/data", {
  signal: AbortSignal.timeout(5000),
});
```

### ✅ Clone before consuming if body is needed multiple times

```javascript
// ✅ Clone before consuming for logging/caching
const response = await fetch(url);
const forCache = response.clone();
const data = await response.json();
cache.put(url, forCache);
```

---

## 13. Bad Practices

### ❌ Ignoring the two-stage design

```javascript
// ❌ Chaining json() directly without checking ok
const data = await fetch("/api/data").then((r) => r.json());
// 404 response: json() succeeds, returns the 404 error body as "data"
```

### ❌ Using XHR when Fetch is available

```javascript
// ❌ Using XHR without a specific reason
const xhr = new XMLHttpRequest();
// XHR for: upload progress (valid), IE support (not needed), legacy code

// ✅ Use Fetch for everything else — cleaner, composable, Promise-based
```

### ❌ Not cancelling fetch on component unmount

```javascript
// ❌ Memory leak: updates state after component is unmounted
useEffect(() => {
  fetch("/api/data")
    .then((r) => r.json())
    .then((data) => setState(data)); // no cleanup!
}, []);
```

---

## 14. Common Mistakes

### Mistake 1 — `fetch()` resolves on HTTP errors

```javascript
// Fetch resolves (not rejects) for 404, 500, etc.
// The error is in response.ok, not a rejection
// See Section 1 — always check response.ok
```

### Mistake 2 — `Content-Type` not set for POST with JSON

```javascript
// ❌ Server may not parse body correctly
fetch("/api/create", {
  method: "POST",
  body: JSON.stringify(data),
  // Missing: Content-Type: application/json
});

// ✅ Always set Content-Type for JSON bodies
fetch("/api/create", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(data),
});
```

### Mistake 3 — Awaiting inside a loop (creating a waterfall)

```javascript
// ❌ Sequential: N × RTT
for (const id of productIds) {
  const product = await fetch(`/api/products/${id}`).then((r) => r.json());
  processProduct(product);
}

// ✅ Parallel: 1 RTT for all
const products = await Promise.all(
  productIds.map((id) => fetch(`/api/products/${id}`).then((r) => r.json())),
);
products.forEach(processProduct);
```

---

## 15. Interview-Level Explanation

> **"How does the Fetch API work? What are the key differences from XHR?"**

**Strong answer:**

> "Fetch is Promise-based and composable, while XHR is callback-based and stateful. The most important conceptual difference is Fetch's two-phase design: the Promise returned by `fetch()` resolves when the response headers arrive — not when the body is downloaded. You get a Response object with status and headers, and you then call `response.json()` or `response.text()` to consume the body. This enables efficient early checks: if the status is 404, you don't need to download the body at all.
>
> The other critical thing to understand is that Fetch never rejects on HTTP errors. A 404 or 500 returns a fulfilled Promise with `response.ok = false`. Only network failures, CORS errors, or DNS failures cause rejection. This is a very common source of bugs — code that assumes rejection means 'something went wrong' misses HTTP-level errors entirely. The fix is always checking `response.ok` before processing the body.
>
> Fetch also supports streaming — `response.body` is a ReadableStream. You can process large responses chunk by chunk without waiting for the full download. Combined with AbortController for cancellation and `AbortSignal.timeout()` for timeouts, you have a complete toolkit.
>
> XHR still has one advantage: upload progress via `xhr.upload.onprogress`. Fetch doesn't expose upload progress in a meaningful way yet, so file upload UIs still often use XHR.
>
> CORS with Fetch is controlled by the `mode` and `credentials` options. The default mode is `cors`, which requires the server to send correct CORS headers. Credentials — cookies and auth headers — are only sent to same-origin by default; you opt in to cross-origin credential sharing with `credentials: 'include'`, which requires the server to respond with `Access-Control-Allow-Credentials: true` and a specific origin in `Access-Control-Allow-Origin` (not the wildcard `*`).
>
> For production use, I wrap Fetch in a client that adds: auth token injection via an interceptor, timeout via AbortSignal, response status validation, retry with exponential backoff for 5xx errors, and typed error handling."

---

## 16. Exercises

### Exercise 1 — Fix the fetch chain

```javascript
// Find all bugs in this fetch implementation:
async function getUser(id) {
  const data = await fetch(`/api/users/${id}`).then((r) => r.json());

  return data;
}

async function updateUser(id, updates) {
  fetch(`/api/users/${id}`, {
    method: "PUT",
    body: JSON.stringify(updates),
  });
}

async function loadUserDashboard(userId) {
  const user = await getUser(userId);
  const orders = await fetch(`/api/orders?userId=${userId}`).then((r) =>
    r.json(),
  );
  const reviews = await fetch(`/api/reviews?userId=${userId}`).then((r) =>
    r.json(),
  );

  return { user, orders, reviews };
}
```

<details>
<summary>Solution</summary>

```javascript
// Bugs found:

// 1. getUser: no response.ok check — returns error body on 404/500
async function getUser(id) {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error(`User not found: ${response.status}`);
  return response.json();
}

// 2. updateUser: fire-and-forget (no await, no error handling, no ok check)
// Also: missing Content-Type header for JSON body
async function updateUser(id, updates) {
  const response = await fetch(`/api/users/${id}`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" }, // ← added
    body: JSON.stringify(updates),
  });
  if (!response.ok) throw new Error(`Update failed: ${response.status}`);
  return response.json();
}

// 3. loadUserDashboard: sequential waterfall (orders waits for user, reviews waits for orders)
// orders and reviews don't depend on each other — run them in parallel
async function loadUserDashboard(userId) {
  const user = await getUser(userId); // must get user first (base data)

  // orders and reviews are independent — run in parallel
  const [orders, reviews] = await Promise.all([
    fetch(`/api/orders?userId=${userId}`).then((r) => {
      if (!r.ok) throw new Error(`Orders failed: ${r.status}`);
      return r.json();
    }),
    fetch(`/api/reviews?userId=${userId}`).then((r) => {
      if (!r.ok) throw new Error(`Reviews failed: ${r.status}`);
      return r.json();
    }),
  ]);

  return { user, orders, reviews };
}

// Additional improvements:
// - Add AbortSignal.timeout for all requests
// - Add retry logic for transient failures
```

</details>

---

## 🔗 Related Topics

- [`networking/01-http-protocols.md`](./01-http-protocols.md) — HTTP that Fetch communicates over
- [`networking/03-websockets-sse.md`](./03-websockets-sse.md) — Real-time alternatives to request/response
- [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md) — Async/await patterns for Fetch
- [`javascript-core/11-promise-internals.md`](../javascript-core/11-promise-internals.md) — Promises that Fetch uses
- [`caching/04-data-caching.md`](../caching/04-data-caching.md) — Caching Fetch responses

---

<div align="center">

**Next:** [`networking/03-websockets-sse.md`](./03-websockets-sse.md) →

</div>
