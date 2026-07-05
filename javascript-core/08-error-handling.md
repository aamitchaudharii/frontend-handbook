# 23 — Error Handling

> **"An unhandled error is a lie — the program says it's still running correctly when it isn't. Good error handling isn't about preventing errors; it's about making failures visible, recoverable, and informative. The worst error is a silent one."**

🟡 **Level: Intermediate**

---

## 📚 Table of Contents

1. [try / catch / finally](#1-try--catch--finally)
2. [Error Types](#2-error-types)
3. [Custom Error Classes](#3-custom-error-classes)
4. [Error Properties](#4-error-properties)
5. [Async Error Handling](#5-async-error-handling)
6. [Global Error Handlers](#6-global-error-handlers)
7. [Error Handling Patterns](#7-error-handling-patterns)
8. [Common Mistakes](#8-common-mistakes)
9. [Exercises](#9-exercises)

---

## 1. try / catch / finally

```javascript
// Basic structure
try {
  // Code that might throw
  const data = JSON.parse(invalidJson); // throws SyntaxError
  processData(data);
} catch (error) {
  // Runs if any error is thrown in the try block
  console.error("Failed to parse:", error.message);
} finally {
  // ALWAYS runs — whether or not an error was thrown
  // Use for cleanup: close connections, release resources
  cleanup();
}

// catch is optional if you have finally
try {
  riskyOperation();
} finally {
  cleanup(); // runs even if riskyOperation throws
}

// finally and return: finally OVERRIDES the try block's return
function example() {
  try {
    return "from try";
  } finally {
    return "from finally"; // ← this wins! 'from try' is discarded
  }
}
example(); // 'from finally'
// ⚠️ Avoid returning in finally — it's confusing

// Rethrowing: catch, handle what you can, rethrow what you can't
try {
  parseConfig(rawConfig);
} catch (error) {
  if (error instanceof SyntaxError) {
    console.error("Config syntax error:", error.message);
    return defaultConfig; // handle and recover
  }
  throw error; // not a syntax error — rethrow for the caller to handle
}
```

---

## 2. Error Types

```javascript
// Built-in error types (all extend Error):

// SyntaxError: invalid JavaScript syntax (usually thrown by parser, not your code)
JSON.parse("invalid json"); // SyntaxError: Unexpected token i

// TypeError: wrong type or operation on wrong type
null.property; // TypeError: Cannot read properties of null
undefined(); // TypeError: undefined is not a function
(42).toUpperCase(); // TypeError: (42).toUpperCase is not a function

// ReferenceError: accessing a variable that doesn't exist
console.log(undeclaredVar); // ReferenceError: undeclaredVar is not defined
// (also thrown in TDZ — accessing let/const before declaration)

// RangeError: value out of allowed range
new Array(-1); // RangeError: Invalid array length
(1.23456789).toFixed(200); // RangeError: toFixed() digits argument must be ≤ 100

// URIError: malformed URI
decodeURI("%"); // URIError: URI malformed

// EvalError: eval() misuse (rare in modern code)

// Checking error type:
try {
  null.foo;
} catch (e) {
  if (e instanceof TypeError) {
    /* handle */
  }
  console.log(e.constructor.name); // 'TypeError'
}
```

---

## 3. Custom Error Classes

```javascript
// Extend Error for domain-specific error types
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;

    // Fix the prototype chain in environments that need it
    Object.setPrototypeOf(this, ValidationError.prototype);
  }
}

class NetworkError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = "NetworkError";
    this.statusCode = statusCode;
    Object.setPrototypeOf(this, NetworkError.prototype);
  }

  get isRetryable() {
    return this.statusCode >= 500 || this.statusCode === 429;
  }
}

class NotFoundError extends NetworkError {
  constructor(resource) {
    super(`${resource} not found`, 404);
    this.name = "NotFoundError";
    this.resource = resource;
    Object.setPrototypeOf(this, NotFoundError.prototype);
  }
}

// Usage:
function validateUser(user) {
  if (!user.email) throw new ValidationError("Email is required", "email");
  if (!user.email.includes("@"))
    throw new ValidationError("Invalid email", "email");
}

// Handling:
try {
  validateUser({ email: "not-an-email" });
} catch (e) {
  if (e instanceof ValidationError) {
    showFieldError(e.field, e.message);
  } else if (e instanceof NetworkError && e.isRetryable) {
    retryRequest();
  } else {
    throw e; // unknown error — propagate
  }
}
```

---

## 4. Error Properties

```javascript
const error = new Error("Something went wrong");

error.message; // 'Something went wrong'
error.name; // 'Error' (or the subclass name)
error.stack; // Full stack trace as a string

// Stack trace example:
// Error: Something went wrong
//   at Object.<anonymous> (/app/index.js:5:13)
//   at Module._compile (internal/modules/cjs/loader.js:1199:30)
//   ...

// Cause (ES2022): chain errors without losing the original
try {
  database.query("SELECT * FROM users");
} catch (original) {
  throw new Error("Failed to load users", { cause: original });
  // error.cause === original (preserves the original error for logging)
}

// Logging errors properly
function logError(error) {
  console.error({
    name: error.name,
    message: error.message,
    stack: error.stack,
    cause: error.cause,
    // any custom properties
    ...Object.fromEntries(
      Object.entries(error).filter(
        ([k]) => !["name", "message", "stack"].includes(k),
      ),
    ),
  });
}
```

---

## 5. Async Error Handling

```javascript
// Promises: .catch() handles rejections in the chain
fetch("/api/data")
  .then((r) => r.json())
  .then(processData)
  .catch((error) => {
    console.error("Request failed:", error);
  });

// async/await: use try/catch
async function loadData() {
  try {
    const response = await fetch("/api/data");
    if (!response.ok) {
      throw new NetworkError(`HTTP Error ${response.status}`, response.status);
    }
    const data = await response.json();
    return data;
  } catch (error) {
    if (error instanceof NetworkError) {
      handleNetworkError(error);
    } else {
      throw error; // propagate non-network errors
    }
  }
}

// Unhandled promise rejection (returns a rejected promise with no .catch())
async function bad() {
  throw new Error("Unhandled!"); // whoever calls bad() must .catch() or await + try/catch
}

// Promise.allSettled vs Promise.all for error handling
const results = await Promise.allSettled([
  fetch("/a"),
  fetch("/b"),
  fetch("/c"),
]);
const [r1, r2, r3] = results;
if (r1.status === "fulfilled") {
  use(r1.value);
}
if (r1.status === "rejected") {
  handleError(r1.reason);
}
// allSettled never throws — all 3 complete regardless of individual failures

// Async error wrapper utility
async function tryCatch(promise) {
  try {
    const data = await promise;
    return [null, data];
  } catch (error) {
    return [error, null];
  }
}
// Usage:
const [error, user] = await tryCatch(fetchUser(id));
if (error) return handleError(error);
use(user);
```

---

## 6. Global Error Handlers

```javascript
// Browser: uncaught synchronous errors
window.onerror = function (message, source, lineno, colno, error) {
  logToMonitoring({ message, source, lineno, colno, stack: error?.stack });
  return true; // suppress browser default error handling
};

// Browser: unhandled promise rejections
window.addEventListener("unhandledrejection", (event) => {
  const error = event.reason;
  logToMonitoring({
    type: "UnhandledRejection",
    message: error?.message,
    stack: error?.stack,
  });
  event.preventDefault(); // suppress default console warning
});

// Node.js equivalents
process.on("uncaughtException", (error) => {
  console.error("Uncaught:", error);
  process.exit(1); // always exit after uncaughtException — state is undefined
});
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection at:", promise, "reason:", reason);
});

// React Error Boundary (for rendering errors)
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  componentDidCatch(error, info) {
    logToMonitoring(error, info.componentStack);
  }
  render() {
    return this.state.hasError ? <ErrorUI /> : this.props.children;
  }
}
```

---

## 7. Error Handling Patterns

### Result type pattern (no exceptions for expected failures)

```typescript
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

async function fetchUser(id: string): Promise<Result<User>> {
  try {
    const user = await api.getUser(id);
    return { ok: true, value: user };
  } catch (e) {
    return { ok: false, error: e as Error };
  }
}

const result = await fetchUser("123");
if (!result.ok) {
  showError(result.error.message);
  return;
}
use(result.value); // TypeScript knows value is User here
```

### Error boundary pattern (React)

```jsx
// Wrap risky sections of the UI in ErrorBoundary components
// See: debugging/04-error-boundaries.md for full implementation
<ErrorBoundary fallback={<ErrorMessage />}>
  <RiskyComponent />
</ErrorBoundary>
```

---

## 8. Common Mistakes

### Mistake 1 — Swallowing errors silently

```javascript
// ❌ Error is caught and discarded — failure is invisible
try {
  processData(input);
} catch (e) {
  // nothing — bug could go unnoticed for days
}

// ✅ At minimum, log the error
try {
  processData(input);
} catch (e) {
  console.error("processData failed:", e);
  reportToMonitoring(e); // send to Sentry, Datadog, etc.
  throw e; // re-throw if you can't recover here
}
```

### Mistake 2 — catch (e) catching too broadly

```javascript
// ❌ Catches AND suppresses ALL errors including bugs you didn't expect
try {
  data = JSON.parse(raw);
  user = findUser(data.userId); // what if this throws a RangeError bug?
  render(user); // what if this throws a TypeError bug?
} catch (e) {
  showError("Parsing failed"); // message may be completely wrong
}

// ✅ Narrow the try block and check error type
let data;
try {
  data = JSON.parse(raw);
} catch (e) {
  if (e instanceof SyntaxError) {
    showError("Invalid JSON");
    return;
  }
  throw e; // unexpected error — re-throw
}
// Use data here, outside the try block
```

### Mistake 3 — Not handling async errors

```javascript
// ❌ Unhandled promise rejection — appears in console, may crash in Node
async function load() {
  throw new Error("Failed");
}
load(); // returns a rejected Promise with no handler

// ✅ Always handle async errors
load().catch(handleError);
// or:
async function safe() {
  try {
    await load();
  } catch (e) {
    handleError(e);
  }
}
```

---

## 9. Exercises

### Exercise 1 — Safe JSON parse

```javascript
// Write a safeJsonParse(str) function that returns [null, parsed] on success
// and [error, null] on failure — no try/catch needed at the call site.
```

<details>
<summary>Solution</summary>

```javascript
function safeJsonParse(str) {
  try {
    return [null, JSON.parse(str)];
  } catch (error) {
    return [error, null];
  }
}

const [err, data] = safeJsonParse('{"name":"Alice"}');
if (err) {
  console.error("Invalid JSON:", err.message);
} else {
  console.log(data.name);
}
```

</details>

---

## 🔗 Related Topics

- [`10-async-patterns.md`](./10-async-patterns.md) — Promise error handling in depth
- [`debugging/04-error-boundaries.md`](../debugging/04-error-boundaries.md) — React error boundaries
