# 04 — Error Boundaries

> **"An error boundary is a promise to your users: 'when part of my application breaks, the rest will keep working.' Without error boundaries, a single uncaught error in a leaf component unmounts the entire tree and shows a blank page. With them, failures are isolated — the broken part shows a graceful fallback, everything else continues."**

Error Boundaries are React's mechanism for catching JavaScript errors in the component tree and rendering a fallback UI instead of crashing the entire application. They're the UI equivalent of a try/catch — but for component rendering, lifecycle methods, and constructors. This document covers the Error Boundary API, placement strategies, error reporting integration, recovery patterns, and the common edge cases where Error Boundaries don't help.

---

## 📚 Table of Contents

1. [What Error Boundaries Are and Aren't](#1-what-error-boundaries-are-and-arent)
2. [Implementing an Error Boundary](#2-implementing-an-error-boundary)
3. [Error Boundary Placement Strategy](#3-error-boundary-placement-strategy)
4. [Error Reporting Integration](#4-error-reporting-integration)
5. [Recovery Patterns](#5-recovery-patterns)
6. [The react-error-boundary Library](#6-the-react-error-boundary-library)
7. [Error Boundaries with Suspense](#7-error-boundaries-with-suspense)
8. [What Error Boundaries Don't Catch](#8-what-error-boundaries-dont-catch)
9. [Custom Error UIs by Context](#9-custom-error-uis-by-context)
10. [Good Practices](#10-good-practices)
11. [Bad Practices](#11-bad-practices)
12. [Common Mistakes](#12-common-mistakes)
13. [Interview-Level Explanation](#13-interview-level-explanation)
14. [Exercises](#14-exercises)

---

## 1. What Error Boundaries Are and Aren't

```
WHAT ERROR BOUNDARIES CATCH:
  ✓ Errors thrown during rendering (in the component function body)
  ✓ Errors thrown in lifecycle methods
  ✓ Errors thrown in constructors of child components
  ✓ Errors thrown in getDerivedStateFromProps

WHAT ERROR BOUNDARIES DO NOT CATCH:
  ✗ Errors in event handlers (onClick, onChange, onSubmit)
     → Use try/catch inside the handler
  ✗ Errors in async code (async/await, Promises, setTimeout)
     → Use try/catch with state for error display
  ✗ Server-side rendering errors
     → SSR has separate error handling
  ✗ Errors in the error boundary itself
     → They bubble up to the nearest parent error boundary

WITHOUT ERROR BOUNDARY:
  Any uncaught error in rendering → React unmounts the ENTIRE component tree
  User sees a blank page or the last rendered state

WITH ERROR BOUNDARY:
  Error is caught at the boundary
  Only the subtree below the boundary is replaced with the fallback
  Everything outside the boundary keeps working
```

---

## 2. Implementing an Error Boundary

Error Boundaries must be class components — they're the one use case where class components remain necessary in modern React:

```tsx
import React, { Component, ErrorInfo, ReactNode } from "react";

interface Props {
  children: ReactNode;
  fallback?: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  onError?: (error: Error, info: ErrorInfo) => void;
  resetKeys?: unknown[]; // reset when any of these values change
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  // Static method: update state when a render error is caught
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  // Instance method: called after render with error info (call stack, component stack)
  componentDidCatch(error: Error, info: ErrorInfo): void {
    // Log to error reporting service
    this.props.onError?.(error, info);
    console.error("ErrorBoundary caught:", error, info.componentStack);
  }

  // Reset when resetKeys change
  componentDidUpdate(prevProps: Props): void {
    if (
      this.state.hasError &&
      this.props.resetKeys &&
      this.props.resetKeys.some((key, i) => key !== prevProps.resetKeys?.[i])
    ) {
      this.reset();
    }
  }

  reset = (): void => {
    this.setState({ hasError: false, error: null });
  };

  render(): ReactNode {
    if (this.state.hasError && this.state.error) {
      const { fallback } = this.props;

      if (typeof fallback === "function") {
        return fallback(this.state.error, this.reset);
      }
      if (fallback) {
        return fallback;
      }

      // Default fallback
      return (
        <div role="alert" style={{ padding: 16, color: "#e53935" }}>
          <h2>Something went wrong.</h2>
          <details>
            <summary>Error details</summary>
            <pre>{this.state.error.message}</pre>
          </details>
          <button onClick={this.reset}>Try Again</button>
        </div>
      );
    }

    return this.props.children;
  }
}

export default ErrorBoundary;
```

### Usage Patterns

```tsx
// Simplest: default fallback
<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>

// Custom static fallback
<ErrorBoundary fallback={<ErrorMessage message="This section failed to load." />}>
  <DataVisualization data={data} />
</ErrorBoundary>

// Dynamic fallback (function): access to error and reset
<ErrorBoundary
  fallback={(error, reset) => (
    <div>
      <p>Failed to load product details: {error.message}</p>
      <button onClick={reset}>Retry</button>
    </div>
  )}
  onError={(error, info) => reportError(error, info)}
>
  <ProductDetail productId={id} />
</ErrorBoundary>

// Reset when route changes (user navigates away and back = fresh start)
const { pathname } = useLocation();
<ErrorBoundary resetKeys={[pathname]}>
  <PageContent />
</ErrorBoundary>
```

---

## 3. Error Boundary Placement Strategy

```
STRATEGY: Place boundaries at meaningful UI isolation points.
  Too few: one error kills large sections of the UI.
  Too many: visual fragmentation — fallbacks everywhere for minor errors.

RECOMMENDED PLACEMENT LEVELS:

1. ROOT (required):
   Catch-all at the very top
   Shows a full-page error state for catastrophic failures
   <ErrorBoundary fallback={<AppCrashPage />}><App /></ErrorBoundary>

2. ROUTE LEVEL:
   Each page/route gets its own boundary
   An error in /products doesn't break /settings
   <Route path="/products" element={
     <ErrorBoundary fallback={<PageError />}><ProductsPage /></ErrorBoundary>
   } />

3. FEATURE/WIDGET LEVEL:
   Isolated features that are independently recoverable
   Sidebar, widget, data visualization, chat, notifications
   <ErrorBoundary fallback={<WidgetError />}><ChatWidget /></ErrorBoundary>

4. DATA BOUNDARY (for dynamic/async data):
   Around components that fetch and display potentially unpredictable data
   Especially useful with Suspense (Section 7)
   <ErrorBoundary fallback={<DataError />}>
     <Suspense fallback={<Spinner />}>
       <UserProfile userId={id} />
     </Suspense>
   </ErrorBoundary>
```

### Visual Layout Example

```
<RootErrorBoundary>            ← catches catastrophic failures
  <App>
    <Header />                  ← no boundary: critical, simple, stable

    <ErrorBoundary>             ← route-level: error in Products doesn't break Settings
      <Route path="/dashboard">
        <ErrorBoundary>         ← widget: error in RevenueChart doesn't break UserTable
          <RevenueChart />
        </ErrorBoundary>
        <ErrorBoundary>         ← widget: independent
          <UserTable />
        </ErrorBoundary>
      </Route>
    </ErrorBoundary>

    <ErrorBoundary>             ← sidebar isolated
      <Sidebar>
        <ErrorBoundary>         ← notifications independent
          <NotificationList />
        </ErrorBoundary>
      </Sidebar>
    </ErrorBoundary>
  </App>
</RootErrorBoundary>
```

---

## 4. Error Reporting Integration

```tsx
// Production error reporting: send to Sentry, Bugsnag, Datadog, etc.

class ErrorBoundary extends Component<Props, State> {
  // ...

  componentDidCatch(error: Error, info: ErrorInfo): void {
    // Report to error tracking service
    if (
      typeof window !== "undefined" &&
      process.env.NODE_ENV === "production"
    ) {
      // Sentry example:
      Sentry.withScope((scope) => {
        scope.setExtras({
          componentStack: info.componentStack,
          // Add additional context:
          userId: getCurrentUserId(),
          route: window.location.pathname,
          timestamp: Date.now(),
        });
        Sentry.captureException(error);
      });
    }

    // Always log in development:
    if (process.env.NODE_ENV === "development") {
      console.error("[ErrorBoundary]", error);
      console.error("[ComponentStack]", info.componentStack);
    }

    this.props.onError?.(error, info);
  }
}

// Initialize Sentry once at app startup (not in every boundary):
// main.tsx
Sentry.init({
  dsn: process.env.VITE_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: __APP_VERSION__,
  tracesSampleRate: 0.1,
});
```

---

## 5. Recovery Patterns

### Pattern 1 — Retry Button

```tsx
// Allow user to retry (re-renders the error'd subtree)
function WidgetWithRetry({ children }: { children: ReactNode }) {
  return (
    <ErrorBoundary
      fallback={(error, reset) => (
        <div className="widget-error">
          <AlertCircleIcon />
          <p>Failed to load this section.</p>
          <button onClick={reset} className="retry-btn">
            Retry
          </button>
        </div>
      )}
    >
      {children}
    </ErrorBoundary>
  );
}
```

### Pattern 2 — Reset on Navigation

```tsx
// Error clears automatically when user navigates to a different route
function RouteErrorBoundary({ children }: { children: ReactNode }) {
  const { pathname } = useLocation();

  return (
    <ErrorBoundary
      resetKeys={[pathname]} // reset whenever pathname changes
      fallback={(error, reset) => (
        <div>
          <h1>Something went wrong on this page</h1>
          <p>{error.message}</p>
          <Link to="/" onClick={reset}>
            Go home
          </Link>
        </div>
      )}
    >
      {children}
    </ErrorBoundary>
  );
}
```

### Pattern 3 — Automatic Retry with Exponential Backoff

```tsx
// For errors likely caused by transient issues (network, server):
function AutoRetryBoundary({ children, maxRetries = 3 }: Props) {
  const [retryCount, setRetryCount] = useState(0);
  const [key, setKey] = useState(0); // force remount on retry

  function handleError(error: Error) {
    if (retryCount < maxRetries) {
      const delay = Math.min(1000 * 2 ** retryCount, 10_000);
      setTimeout(() => {
        setRetryCount((c) => c + 1);
        setKey((k) => k + 1); // new key = unmount + remount = fresh render
      }, delay);
    }
  }

  if (retryCount >= maxRetries) {
    return <FinalErrorFallback />;
  }

  return (
    <ErrorBoundary key={key} onError={handleError}>
      {children}
    </ErrorBoundary>
  );
}
```

### Pattern 4 — Forcing Reset via Key

```tsx
// Changing the key prop forces React to unmount + remount the ErrorBoundary
// (and its children), effectively resetting the error state

function ResetByKey({ children }: { children: ReactNode }) {
  const [resetKey, setResetKey] = useState(0);

  return (
    <ErrorBoundary
      key={resetKey} // changing this unmounts + remounts the boundary tree
      fallback={() => (
        <button onClick={() => setResetKey((k) => k + 1)}>
          Reset and try again
        </button>
      )}
    >
      {children}
    </ErrorBoundary>
  );
}
```

---

## 6. The react-error-boundary Library

The `react-error-boundary` package provides a well-tested Error Boundary with a hooks-compatible API:

```tsx
import { ErrorBoundary, useErrorBoundary } from "react-error-boundary";

// Component with reset functionality:
function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert">
      <p>Something went wrong:</p>
      <pre style={{ whiteSpace: "normal" }}>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

// Usage:
<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onReset={(details) => {
    // optional: reset app state related to this boundary
    console.log("Reset triggered by:", details.reason);
  }}
  resetKeys={[userId]} // auto-reset when userId changes
>
  <UserDashboard />
</ErrorBoundary>;

// useErrorBoundary hook: throw errors INTO an error boundary from async code
function DataLoader({ url }) {
  const { showBoundary } = useErrorBoundary();

  useEffect(() => {
    fetch(url)
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then(setData)
      .catch(showBoundary); // ← sends async error to the nearest ErrorBoundary!
  }, [url, showBoundary]);
  // Without showBoundary: async errors bypass error boundaries entirely
  // With showBoundary: async errors are caught and shown via the boundary
}
```

---

## 7. Error Boundaries with Suspense

Error Boundaries and Suspense pair naturally — Suspense handles the loading state, ErrorBoundary handles the error state:

```tsx
// The canonical pattern for async data fetching with full state handling:
<ErrorBoundary
  fallback={(error) => (
    <div className="data-error">
      <p>Failed to load: {error.message}</p>
    </div>
  )}
>
  <Suspense fallback={<LoadingSpinner />}>
    {/* DataComponent: throws a Promise while loading (Suspense catches it)
        throws an Error on failure (ErrorBoundary catches it) */}
    <DataComponent />
  </Suspense>
</ErrorBoundary>;

// React 18 + TanStack Query: throwOnError makes queries work with Suspense
const { data } = useQuery({
  queryKey: ["user", userId],
  queryFn: () => usersApi.get(userId),
  throwOnError: true, // throws into the nearest ErrorBoundary on failure
  suspense: true, // suspends until data is ready
});
```

---

## 8. What Error Boundaries Don't Catch

```tsx
// ❌ Errors in event handlers — NOT caught by ErrorBoundary
function Component() {
  function handleClick() {
    throw new Error("This is not caught by ErrorBoundary!");
    // → Uncaught error in console, but page doesn't crash via ErrorBoundary
  }
  return <button onClick={handleClick}>Click</button>;
}

// ✅ Handle event handler errors with try/catch + state
function Component() {
  const [error, setError] = useState<Error | null>(null);

  function handleClick() {
    try {
      riskyOperation();
    } catch (err) {
      setError(err as Error);
    }
  }

  if (error)
    return <ErrorMessage error={error} onRetry={() => setError(null)} />;
  return <button onClick={handleClick}>Click</button>;
}

// ❌ Errors in async code — NOT caught by ErrorBoundary
function Component() {
  useEffect(() => {
    async function load() {
      throw new Error("Async errors bypass ErrorBoundary by default");
    }
    load(); // rejected promise — not caught by ErrorBoundary
  }, []);
}

// ✅ Explicitly rethrow into boundary via useErrorBoundary or state
function Component() {
  const { showBoundary } = useErrorBoundary(); // from react-error-boundary

  useEffect(() => {
    async function load() {
      try {
        await fetchData();
      } catch (err) {
        showBoundary(err); // manually sends error to the nearest ErrorBoundary
      }
    }
    load();
  }, [showBoundary]);
}
```

---

## 9. Custom Error UIs by Context

```tsx
// Different UIs for different contexts in the application

// Full-page error (root level):
function AppCrashPage({ error }: { error: Error }) {
  return (
    <div className="app-crash">
      <h1>We're having some trouble</h1>
      <p>The application encountered an unexpected error.</p>
      <button onClick={() => window.location.reload()}>Refresh Page</button>
      {process.env.NODE_ENV === "development" && (
        <details>
          <summary>Technical details</summary>
          <pre>{error.stack}</pre>
        </details>
      )}
    </div>
  );
}

// Section/widget error (inline):
function SectionError({ onRetry }: { onRetry: () => void }) {
  return (
    <div className="section-error" role="alert">
      <AlertIcon />
      <span>This section couldn't load.</span>
      <button onClick={onRetry}>Try again</button>
    </div>
  );
}

// Data card error (compact):
function CardError() {
  return (
    <div className="card card--error">
      <span>⚠️ Error loading data</span>
    </div>
  );
}

// Application:
<ErrorBoundary fallback={<AppCrashPage />}>
  <App>
    <ErrorBoundary fallback={(_, reset) => <SectionError onRetry={reset} />}>
      <DataSection />
    </ErrorBoundary>
    <ErrorBoundary fallback={<CardError />}>
      <MetricCard />
    </ErrorBoundary>
  </App>
</ErrorBoundary>;
```

---

## 10. Good Practices

### ✅ Always have a root-level error boundary

```tsx
// Every React app should have this:
ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <ErrorBoundary fallback={<AppCrashPage />}>
      <App />
    </ErrorBoundary>
  </React.StrictMode>,
);
```

### ✅ Report errors to your monitoring service in componentDidCatch

```tsx
// Every production ErrorBoundary should report errors
componentDidCatch(error: Error, info: ErrorInfo) {
  Sentry.captureException(error, { extra: { componentStack: info.componentStack } });
}
```

### ✅ Show dev-only error details, clean UI in production

```tsx
// ✅ Different content for different environments
{
  process.env.NODE_ENV === "development" ? (
    <pre>{error.stack}</pre> // full stack in dev
  ) : (
    <p>Something went wrong.</p> // clean message in prod
  );
}
```

---

## 11. Bad Practices

### ❌ One global error boundary at the root only

```tsx
// ❌ Too coarse: any error shows the app crash page
<ErrorBoundary fallback={<AppCrashPage />}>
  <App />
</ErrorBoundary>

// A toast notification failing to render: whole app crashes to the crash page
// ✅ Add granular boundaries so toast errors don't crash the app
```

### ❌ Empty catch blocks in componentDidCatch

```tsx
// ❌ Silent errors: crashes are invisible in production
componentDidCatch(error: Error) {
  // nothing here — errors are swallowed silently
}
// ✅ Always log/report
componentDidCatch(error: Error, info: ErrorInfo) {
  reportError(error, { context: info.componentStack });
}
```

---

## 12. Common Mistakes

### Mistake 1 — Using Error Boundaries for event handler errors

```tsx
// ❌ Expecting ErrorBoundary to catch errors in onClick
<ErrorBoundary fallback={<Error />}>
  <button
    onClick={() => {
      throw new Error();
    }}
  >
    Click
  </button>
</ErrorBoundary>;
// Error does NOT get caught by the boundary — it's an event handler

// ✅ Use try/catch inside event handlers
function handleClick() {
  try {
    riskyOperation();
  } catch (err) {
    setError(err);
  }
}
```

### Mistake 2 — Not resetting error state after navigation

```tsx
// ❌ User gets stuck on the error page even after navigating away and back
<ErrorBoundary fallback={<Error />}>
  <RouteContent />
</ErrorBoundary>
// The error is still in ErrorBoundary's state even after the route changes

// ✅ Reset on route change via resetKeys
<ErrorBoundary resetKeys={[currentPath]} fallback={<Error />}>
  <RouteContent />
</ErrorBoundary>
```

### Mistake 3 — Swallowing errors in production without reporting

```tsx
// ❌ Errors caught but never reported — you have no visibility into production crashes
class ErrorBoundary extends Component {
  componentDidCatch() {} // empty — errors silently disappear
  render() {
    if (this.state.hasError) return <Fallback />;
    return this.props.children;
  }
}
// ✅ componentDidCatch should ALWAYS report to error monitoring in production
```

---

## 13. Interview-Level Explanation

> **"What are Error Boundaries in React? When and how do you use them?"**

**Strong answer:**

> "Error Boundaries are React's mechanism for catching JavaScript errors in the rendering phase of the component tree and showing a fallback UI instead of crashing the application. Without them, any uncaught rendering error causes React to unmount the entire component tree, leaving users with a blank page. With Error Boundaries, errors are isolated: only the subtree below the boundary is replaced with the fallback — everything outside continues working normally.
>
> They're class components because they use `getDerivedStateFromError` — a static lifecycle method that returns state when an error is caught — and `componentDidCatch`, which receives the error and a component stack trace. Custom hooks can't implement these lifecycle methods, which is the one remaining use case for class components in modern React.
>
> Important limitation: Error Boundaries only catch errors during rendering, in lifecycle methods, and in constructors. They don't catch errors in event handlers — those need try/catch inside the handler — and they don't catch async errors from Promises or async/await. The `react-error-boundary` library provides a `useErrorBoundary` hook that lets you manually send async errors to the nearest boundary, which is how you handle the async case.
>
> For placement strategy, I use a layered approach. The root level gets a catch-all boundary that shows a full-page crash screen for catastrophic failures. Each route gets its own boundary so an error on the Products page doesn't crash the Settings page. Independent widgets — a chart, a data table, a notification sidebar — each get their own boundary so widget-level failures show a compact 'couldn't load' message without affecting the rest of the page.
>
> For recovery, the `resetKeys` prop is the most practical pattern: when its values change (like the current route pathname changing), the boundary resets its error state, so the user gets a fresh render when they navigate away and back. A retry button resets via the `reset` callback. The `componentDidCatch` method should always report to error monitoring like Sentry in production — without this, production crashes are completely invisible to the engineering team."

---

## 14. Exercises

### Exercise 1 — Build a production-ready Error Boundary

Build an Error Boundary that:

- Catches render errors and shows a context-appropriate fallback
- Reports to a monitoring service in production
- Supports retry (reset) on demand
- Resets automatically when `resetKey` prop changes
- Shows detailed error info only in development

<details>
<summary>Solution</summary>

```tsx
import { Component, ErrorInfo, ReactNode } from "react";

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: "full-page" | "inline" | "compact" | ReactNode;
  onError?: (error: Error, info: ErrorInfo) => void;
  resetKey?: unknown;
}

interface ErrorBoundaryState {
  error: Error | null;
}

export class ErrorBoundary extends Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = { error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { error };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    // 1. Report to monitoring
    if (process.env.NODE_ENV === "production") {
      reportError(error, {
        componentStack: info.componentStack,
        boundaryProps: { fallback: this.props.fallback },
        url: window.location.href,
      });
    }

    // 2. Log in development
    console.error("[ErrorBoundary]", error.message);
    console.error(info.componentStack);

    // 3. Notify parent if needed
    this.props.onError?.(error, info);
  }

  componentDidUpdate(prevProps: ErrorBoundaryProps): void {
    if (this.state.error && this.props.resetKey !== prevProps.resetKey) {
      this.setState({ error: null });
    }
  }

  reset = (): void => {
    this.setState({ error: null });
  };

  render(): ReactNode {
    const { error } = this.state;
    if (!error) return this.props.children;

    const { fallback } = this.props;

    // Custom ReactNode fallback
    if (typeof fallback !== "string" && fallback != null) return fallback;

    // Full-page: for root-level boundary
    if (!fallback || fallback === "full-page") {
      return (
        <div className="error-page" role="alert">
          <h1>Something went wrong</h1>
          <p>The application encountered an unexpected error.</p>
          <button onClick={this.reset}>Try again</button>
          <button onClick={() => (window.location.href = "/")}>Go home</button>
          {process.env.NODE_ENV === "development" && (
            <details>
              <summary>Developer details</summary>
              <pre>{error.stack}</pre>
            </details>
          )}
        </div>
      );
    }

    // Inline: for page/section boundaries
    if (fallback === "inline") {
      return (
        <div className="error-section" role="alert">
          <p>⚠️ This section couldn't load.</p>
          <button onClick={this.reset}>Retry</button>
          {process.env.NODE_ENV === "development" && (
            <code>{error.message}</code>
          )}
        </div>
      );
    }

    // Compact: for widget boundaries
    return (
      <div className="error-compact" role="alert" title={error.message}>
        ⚠️ Error
      </div>
    );
  }
}

// Utility: mock error reporting function
function reportError(error: Error, context: Record<string, unknown>) {
  // In real app: Sentry.captureException(error, { extra: context });
  console.error("[ERROR REPORTED]", error.message, context);
}

// Usage:
export default function App() {
  return (
    <ErrorBoundary fallback="full-page">
      <Layout>
        <ErrorBoundary fallback="inline" resetKey={currentRoute}>
          <MainContent />
        </ErrorBoundary>
        <ErrorBoundary fallback="compact">
          <Sidebar />
        </ErrorBoundary>
      </Layout>
    </ErrorBoundary>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`debugging/03-debugging-strategies.md`](./03-debugging-strategies.md) — General debugging techniques
- [`anti-patterns/05-memory-leaks.md`](../anti-patterns/05-memory-leaks.md) — Errors from leaks
- [`testing/02-integration-testing.md`](../testing/02-integration-testing.md) — Testing error boundary behavior

---

<div align="center">

**`debugging/` section complete!** 🎉

All 4 debugging files done:
`01-chrome-devtools.md` · `02-react-devtools.md` · `03-debugging-strategies.md` · **`04-error-boundaries.md`** ✓

**Next section:** [`interview/`](../interview/) →

</div>
