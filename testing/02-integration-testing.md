# 02 — Integration Testing

> **"Integration tests are where you find the bugs that unit tests miss. Unit tests verify your functions work in isolation. Integration tests verify that when you wire them together, the seams hold."**

Integration tests occupy the most valuable position in the testing pyramid — they give you significantly more confidence than unit tests at a fraction of the cost of E2E tests. They test how modules, components, and systems interact, without the overhead of a real browser or network. This document covers integration testing philosophy, React Testing Library patterns, API mocking strategies, component integration, and how to write tests that remain meaningful as your codebase evolves.

---

## 📚 Table of Contents

1. [What Integration Testing Is](#1-what-integration-testing-is)
2. [The Testing Philosophy — Test Behavior, Not Implementation](#2-the-testing-philosophy--test-behavior-not-implementation)
3. [React Testing Library — The Right Approach](#3-react-testing-library--the-right-approach)
4. [Setup and Configuration](#4-setup-and-configuration)
5. [Querying Elements — The Priority Order](#5-querying-elements--the-priority-order)
6. [User Interactions with userEvent](#6-user-interactions-with-userevent)
7. [Async Testing Patterns](#7-async-testing-patterns)
8. [Mocking APIs — MSW (Mock Service Worker)](#8-mocking-apis--msw-mock-service-worker)
9. [Testing Forms](#9-testing-forms)
10. [Testing Routing and Navigation](#10-testing-routing-and-navigation)
11. [Testing with Context and Providers](#11-testing-with-context-and-providers)
12. [Testing Custom Hooks](#12-testing-custom-hooks)
13. [Snapshot Testing — When and How](#13-snapshot-testing--when-and-how)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. What Integration Testing Is

Integration tests verify that multiple units work correctly together. They sit between unit tests (isolated function testing) and E2E tests (full browser automation).

```
What integration tests cover:

Unit tests verify:
  add(2, 3) === 5
  validateEmail('bad') === false

Integration tests verify:
  LoginForm renders → user fills in credentials → submits →
  calls the auth API → on success navigates to dashboard

E2E tests verify:
  Real browser opens the app → real user journey from login to checkout
```

### The Integration Test Sweet Spot

```
Test scope comparison:

Unit test (too narrow):
  → Tests authService.login() in isolation with mocked fetch
  → Doesn't verify LoginForm calls authService correctly
  → Doesn't verify navigation happens after success
  → Multiple unit tests could all pass while the feature is broken

E2E test (too broad):
  → Runs a real browser
  → Takes 5-30 seconds per test
  → Flaky due to network, timing, environment
  → Expensive to run at scale

Integration test (just right):
  → Renders the real LoginForm component
  → Uses real form validation, real state management
  → Mocks only the HTTP request (not the entire auth service)
  → Verifies the component navigates after success
  → Runs in ~100ms, deterministic, no browser needed
```

---

## 2. The Testing Philosophy — Test Behavior, Not Implementation

This principle, from Kent C. Dodds' Testing Library philosophy, is the most important concept in integration testing:

### The Wrong Approach — Testing Implementation

```javascript
// ❌ Testing implementation details — fragile
test("LoginForm internal state", () => {
  const wrapper = shallow(<LoginForm />);

  // Testing component internals
  expect(wrapper.state("email")).toBe("");
  expect(wrapper.state("isLoading")).toBe(false);

  // Calling internal methods directly
  wrapper.instance().handleEmailChange({ target: { value: "test@test.com" } });
  expect(wrapper.state("email")).toBe("test@test.com");
});

// Problems:
// - breaks if you rename state from 'email' to 'emailInput'
// - breaks if you refactor to hooks (no .state())
// - breaks if you change the handler name
// - tests the HOW, not the WHAT
```

### The Right Approach — Testing Behavior

```javascript
// ✅ Testing behavior — what users actually experience
test("user can log in with valid credentials", async () => {
  render(<LoginForm />);

  // Interact as a user would
  await userEvent.type(screen.getByLabelText("Email"), "user@example.com");
  await userEvent.type(screen.getByLabelText("Password"), "password123");
  await userEvent.click(screen.getByRole("button", { name: "Sign In" }));

  // Assert what the user sees
  expect(screen.getByText("Welcome back!")).toBeInTheDocument();
});

// Benefits:
// - survives any internal refactoring
// - tests the actual user experience
// - catches real bugs (broken rendering, missing event handlers)
// - serves as documentation of expected behavior
```

### The Guiding Principle

> **"The more your tests resemble the way your software is used, the more confidence they can give you."** — Kent C. Dodds

---

## 3. React Testing Library — The Right Approach

React Testing Library (RTL) is the standard for React integration testing. It renders components into a real DOM (jsdom) and provides queries that mirror how users find elements.

```
RTL philosophy:
  → Render the real component tree (not shallow rendering)
  → Query elements the way users find them (by label, role, text)
  → Interact as users do (type, click, submit)
  → Assert what users see (text, ARIA state, visibility)
  → Avoid testing implementation details (state, refs, methods)
```

### Why Not Enzyme (Shallow Rendering)?

```javascript
// ❌ Enzyme shallow rendering — tests the wrong things
import { shallow } from "enzyme";

test("renders button", () => {
  const wrapper = shallow(<LoginForm onSubmit={jest.fn()} />);
  expect(wrapper.find("button").text()).toBe("Sign In");
  // This doesn't test if the button actually works
  // It doesn't test if it's accessible
  // It doesn't test if clicking it triggers onSubmit
});

// ✅ RTL — tests what matters
test("renders and submits form", async () => {
  const onSubmit = jest.fn();
  render(<LoginForm onSubmit={onSubmit} />);

  await userEvent.type(screen.getByLabelText("Email"), "test@test.com");
  await userEvent.click(screen.getByRole("button", { name: "Sign In" }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: "test@test.com",
    password: "",
  });
});
```

---

## 4. Setup and Configuration

### Installation

```bash
npm install --save-dev \
  @testing-library/react \
  @testing-library/user-event \
  @testing-library/jest-dom \
  jest \
  jest-environment-jsdom \
  @types/jest # if using TypeScript
```

### `jest.config.js`

```javascript
/** @type {import('jest').Config} */
module.exports = {
  testEnvironment: "jsdom", // simulate browser DOM
  setupFilesAfterFramework: ["<rootDir>/jest.setup.js"],
  moduleNameMapper: {
    // Handle CSS imports (if using CSS modules)
    "\\.css$": "identity-obj-proxy",
    // Handle image/file imports
    "\\.(jpg|jpeg|png|gif|svg)$": "<rootDir>/__mocks__/fileMock.js",
    // Alias paths (if using them)
    "^@/(.*)$": "<rootDir>/src/$1",
  },
  transform: {
    "^.+\\.(ts|tsx)$": ["ts-jest", { tsconfig: "tsconfig.json" }],
    "^.+\\.(js|jsx)$": "babel-jest",
  },
  collectCoverageFrom: [
    "src/**/*.{ts,tsx,js,jsx}",
    "!src/**/*.d.ts",
    "!src/index.tsx",
  ],
};
```

### `jest.setup.js`

```javascript
// Import custom matchers (toBeInTheDocument, toHaveTextContent, etc.)
import "@testing-library/jest-dom";

// Polyfill for environments that don't have ResizeObserver
global.ResizeObserver = class {
  observe() {}
  unobserve() {}
  disconnect() {}
};

// Suppress specific warnings that clutter test output
const originalError = console.error;
beforeAll(() => {
  console.error = (...args) => {
    // Suppress known non-critical warnings
    if (
      typeof args[0] === "string" &&
      args[0].includes("Warning: ReactDOM.render")
    ) {
      return;
    }
    originalError.call(console, ...args);
  };
});
afterAll(() => {
  console.error = originalError;
});
```

### Custom Render Helper

Wrap components with providers that every component needs:

```typescript
// test-utils.tsx — reusable render with all providers
import React from 'react';
import { render, RenderOptions } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ThemeProvider } from './contexts/ThemeContext';
import { AuthProvider } from './contexts/AuthContext';

function AllProviders({ children }: { children: React.ReactNode }) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },   // don't retry in tests
      mutations: { retry: false },
    },
  });

  return (
    <BrowserRouter>
      <QueryClientProvider client={queryClient}>
        <ThemeProvider theme="light">
          <AuthProvider>
            {children}
          </AuthProvider>
        </ThemeProvider>
      </QueryClientProvider>
    </BrowserRouter>
  );
}

const customRender = (ui: React.ReactElement, options?: RenderOptions) =>
  render(ui, { wrapper: AllProviders, ...options });

// Re-export everything from RTL
export * from '@testing-library/react';
// Override render with our version
export { customRender as render };
```

```typescript
// Using the custom render in tests
import { render, screen } from '../test-utils'; // NOT from @testing-library/react

test('product page loads', async () => {
  render(<ProductPage productId="123" />);
  // All providers already wrapped — no need to add them per test
});
```

---

## 5. Querying Elements — The Priority Order

RTL provides multiple ways to query DOM elements. Use them in this priority order:

### Priority 1 — Accessible Queries (Best)

```typescript
// By role — mirrors ARIA and semantic HTML
screen.getByRole("button", { name: "Submit" });
screen.getByRole("textbox", { name: "Email" }); // <input type="text">
screen.getByRole("heading", { level: 1 });
screen.getByRole("link", { name: "Go to dashboard" });
screen.getByRole("checkbox", { name: "Remember me" });
screen.getByRole("combobox", { name: "Country" }); // <select>
screen.getByRole("dialog", { name: "Confirm Delete" });

// By label text — for labeled form inputs
screen.getByLabelText("Email Address");
screen.getByLabelText("Password");

// By placeholder
screen.getByPlaceholderText("Search products...");

// By text content
screen.getByText("Welcome back, Alice");
screen.getByText(/submit/i); // case-insensitive regex
```

### Priority 2 — Semantic Queries

```typescript
// By alt text (images)
screen.getByAltText("Company logo");

// By title
screen.getByTitle("Close dialog");

// By display value (current value of input/select/textarea)
screen.getByDisplayValue("user@example.com");
```

### Priority 3 — Test IDs (When All Else Fails)

```typescript
// By test ID — explicit marker in the component
screen.getByTestId("product-card");

// HTML: <div data-testid="product-card">
// Use sparingly — prefer semantic queries
```

### Query Variants

```typescript
// getBy*: throws if not found or multiple found
const button = screen.getByRole("button", { name: "Submit" });

// queryBy*: returns null if not found (use for "not present" assertions)
const error = screen.queryByRole("alert");
expect(error).not.toBeInTheDocument();

// findBy*: async, returns promise, retries until found or timeout
const result = await screen.findByText("Data loaded");

// getAllBy*: returns array, throws if none found
const items = screen.getAllByRole("listitem");
expect(items).toHaveLength(3);

// queryAllBy*: returns array, returns [] if none found
const warnings = screen.queryAllByRole("alert");

// findAllBy*: async array version
const rows = await screen.findAllByRole("row");
```

---

## 6. User Interactions with userEvent

`@testing-library/user-event` simulates real user interactions more accurately than `fireEvent`.

```typescript
import userEvent from "@testing-library/user-event";

// Always create a user instance at the start of each test
const user = userEvent.setup();

// Typing
await user.type(screen.getByLabelText("Email"), "test@example.com");
// Fires: focus, keydown, keypress, input, keyup for each character

// Clearing and retyping
await user.clear(screen.getByLabelText("Email"));
await user.type(screen.getByLabelText("Email"), "new@email.com");

// Click
await user.click(screen.getByRole("button", { name: "Submit" }));
// Fires: pointerover, pointerenter, mouseover, mouseenter,
//        pointermove, mousemove, pointerdown, mousedown,
//        focus, pointerup, mouseup, click

// Double click
await user.dblClick(element);

// Keyboard interactions
await user.keyboard("{Enter}");
await user.keyboard("{Tab}");
await user.keyboard("{Escape}");
await user.keyboard("{ArrowDown}");

// Select option in dropdown
await user.selectOptions(
  screen.getByRole("combobox", { name: "Country" }),
  "United States",
);

// Check/uncheck checkbox
await user.click(screen.getByRole("checkbox", { name: "Remember me" }));

// Upload file
const file = new File(["content"], "test.pdf", { type: "application/pdf" });
await user.upload(screen.getByLabelText("Upload document"), file);

// Hover
await user.hover(screen.getByRole("button", { name: "Delete" }));
await user.unhover(screen.getByRole("button", { name: "Delete" }));
```

### `fireEvent` vs `userEvent`

```typescript
// fireEvent: dispatches a single event (lower-level)
import { fireEvent } from "@testing-library/react";
fireEvent.click(button); // only fires the 'click' event

// userEvent: simulates the full sequence of events a real user generates
import userEvent from "@testing-library/user-event";
await userEvent.click(button); // fires pointer events, mouse events, focus, click

// Use userEvent by default — it catches more bugs
// Use fireEvent for edge cases or when testing specific events
```

---

## 7. Async Testing Patterns

Most integration tests involve async behavior — API calls, state updates, animations.

### Waiting for Elements

```typescript
import { render, screen, waitFor } from '@testing-library/react';

// findBy*: the right tool for elements that appear asynchronously
test('shows data after loading', async () => {
  render(<UserProfile userId="42" />);

  // Loading state appears immediately
  expect(screen.getByRole('progressbar')).toBeInTheDocument();

  // Wait for data to appear (retries until present or timeout)
  const name = await screen.findByText('Alice Smith');
  expect(name).toBeInTheDocument();

  // Loading state should be gone
  expect(screen.queryByRole('progressbar')).not.toBeInTheDocument();
});
```

### `waitFor` for Custom Assertions

```typescript
import { waitFor } from '@testing-library/react';

test('form shows success message after submit', async () => {
  const user = userEvent.setup();
  render(<ContactForm />);

  await user.type(screen.getByLabelText('Name'), 'Alice');
  await user.click(screen.getByRole('button', { name: 'Send' }));

  // waitFor: retry assertion until it passes or timeout
  await waitFor(() => {
    expect(screen.getByText('Message sent!')).toBeInTheDocument();
  });

  // Multiple assertions in one waitFor
  await waitFor(() => {
    expect(screen.queryByRole('progressbar')).not.toBeInTheDocument();
    expect(screen.getByRole('alert')).toHaveTextContent('Message sent!');
  });
});
```

### Testing Loading, Error, and Success States

```typescript
test('product list shows all states', async () => {
  // Mock: initial loading then success
  server.use(
    http.get('/api/products', async () => {
      await delay(100); // simulate network delay
      return HttpResponse.json(mockProducts);
    })
  );

  render(<ProductList />);

  // Loading state
  expect(screen.getByRole('status', { name: /loading/i })).toBeInTheDocument();

  // Success state
  await screen.findByRole('list', { name: 'Products' });
  expect(screen.getAllByRole('listitem')).toHaveLength(mockProducts.length);
  expect(screen.queryByRole('status', { name: /loading/i })).not.toBeInTheDocument();
});

test('product list shows error state', async () => {
  server.use(
    http.get('/api/products', () =>
      new HttpResponse(null, { status: 500 })
    )
  );

  render(<ProductList />);

  await screen.findByRole('alert');
  expect(screen.getByRole('alert')).toHaveTextContent('Failed to load products');
  expect(screen.getByRole('button', { name: 'Try Again' })).toBeInTheDocument();
});
```

---

## 8. Mocking APIs — MSW (Mock Service Worker)

Mock Service Worker (MSW) intercepts network requests at the service worker level — the same place real network requests go. Your component code doesn't change; only the server responds differently in tests.

### Why MSW Over jest.mock(fetch)

```
jest.mock(fetch) problems:
  → Mocking at the wrong layer (mocks the transport, not the API)
  → Component uses fetch, React Query, axios, SWR — each needs different mock
  → Doesn't test that your request shape is correct
  → Hard to simulate different HTTP status codes

MSW advantages:
  → Intercepts at network level — works with any HTTP client
  → Same handlers work for unit tests AND the browser (shared mocks)
  → Tests the actual HTTP request your code sends
  → Realistic: can simulate delays, errors, partial responses
```

### MSW Setup

```typescript
// src/mocks/handlers.ts — API request handlers
import { http, HttpResponse, delay } from "msw";

export const handlers = [
  // GET /api/users/:id
  http.get("/api/users/:id", ({ params }) => {
    const user = {
      id: params.id,
      name: "Alice Smith",
      email: "alice@example.com",
      role: "admin",
    };
    return HttpResponse.json(user);
  }),

  // GET /api/products
  http.get("/api/products", () =>
    HttpResponse.json([
      { id: 1, name: "Widget Pro", price: 29.99 },
      { id: 2, name: "Gadget Max", price: 49.99 },
    ]),
  ),

  // POST /api/contact
  http.post("/api/contact", async ({ request }) => {
    const body = await request.json();
    // Validate required fields
    if (!body.name || !body.email) {
      return HttpResponse.json(
        { error: "name and email are required" },
        { status: 400 },
      );
    }
    return HttpResponse.json(
      { id: "msg_123", status: "sent" },
      { status: 201 },
    );
  }),
];
```

```typescript
// src/mocks/server.ts — test server
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

```typescript
// jest.setup.js — start server for all tests
import { server } from "./src/mocks/server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers()); // reset overrides after each test
afterAll(() => server.close());
```

### Using MSW in Tests

```typescript
import { server } from '../mocks/server';
import { http, HttpResponse } from 'msw';

test('renders user profile', async () => {
  render(<UserProfile userId="42" />);

  // The default handler returns Alice — no override needed
  await screen.findByText('Alice Smith');
  expect(screen.getByText('admin')).toBeInTheDocument();
});

test('handles user not found', async () => {
  // Override handler for this test only
  server.use(
    http.get('/api/users/:id', () =>
      new HttpResponse(null, { status: 404 })
    )
  );

  render(<UserProfile userId="999" />);

  await screen.findByRole('alert');
  expect(screen.getByRole('alert')).toHaveTextContent('User not found');
});

test('handles slow network', async () => {
  server.use(
    http.get('/api/users/:id', async () => {
      await delay(2000); // 2 second delay
      return HttpResponse.json({ id: '42', name: 'Alice' });
    })
  );

  render(<UserProfile userId="42" />);

  // Should show loading state during delay
  expect(screen.getByRole('progressbar')).toBeInTheDocument();
  // Eventually resolves
  await screen.findByText('Alice', {}, { timeout: 3000 });
});
```

---

## 9. Testing Forms

Forms are one of the highest-value areas for integration testing.

### Basic Form Test

```typescript
import { render, screen } from '../test-utils';
import userEvent from '@testing-library/user-event';
import { server } from '../mocks/server';
import { http, HttpResponse } from 'msw';

describe('ContactForm', () => {
  const user = userEvent.setup();

  test('submits form with valid data', async () => {
    let capturedBody: unknown;

    server.use(
      http.post('/api/contact', async ({ request }) => {
        capturedBody = await request.json();
        return HttpResponse.json({ status: 'sent' }, { status: 201 });
      })
    );

    render(<ContactForm />);

    await user.type(screen.getByLabelText('Name'), 'Alice Smith');
    await user.type(screen.getByLabelText('Email'), 'alice@example.com');
    await user.type(screen.getByLabelText('Message'), 'Hello, I have a question.');

    await user.click(screen.getByRole('button', { name: 'Send Message' }));

    // Verify what was sent to the server
    expect(capturedBody).toEqual({
      name:    'Alice Smith',
      email:   'alice@example.com',
      message: 'Hello, I have a question.',
    });

    // Verify success feedback
    await screen.findByRole('alert');
    expect(screen.getByRole('alert')).toHaveTextContent('Message sent!');

    // Form should reset after success
    expect(screen.getByLabelText('Name')).toHaveValue('');
    expect(screen.getByLabelText('Email')).toHaveValue('');
  });

  test('shows validation errors for empty required fields', async () => {
    render(<ContactForm />);

    // Submit without filling in anything
    await user.click(screen.getByRole('button', { name: 'Send Message' }));

    // Validation errors should appear
    expect(screen.getByText('Name is required')).toBeInTheDocument();
    expect(screen.getByText('Email is required')).toBeInTheDocument();

    // Form was not submitted (no loading state, no success message)
    expect(screen.queryByRole('progressbar')).not.toBeInTheDocument();
    expect(screen.queryByText('Message sent!')).not.toBeInTheDocument();
  });

  test('shows server error without clearing form', async () => {
    server.use(
      http.post('/api/contact', () =>
        HttpResponse.json({ error: 'Service unavailable' }, { status: 503 })
      )
    );

    render(<ContactForm />);

    await user.type(screen.getByLabelText('Name'), 'Alice');
    await user.type(screen.getByLabelText('Email'), 'alice@example.com');
    await user.click(screen.getByRole('button', { name: 'Send Message' }));

    // Error should appear
    await screen.findByRole('alert');
    expect(screen.getByRole('alert')).toHaveTextContent('Failed to send. Please try again.');

    // Form data preserved (user can fix and resubmit)
    expect(screen.getByLabelText('Name')).toHaveValue('Alice');
    expect(screen.getByLabelText('Email')).toHaveValue('alice@example.com');
  });

  test('disables submit button while submitting', async () => {
    server.use(
      http.post('/api/contact', async () => {
        await delay(500);
        return HttpResponse.json({ status: 'sent' });
      })
    );

    render(<ContactForm />);

    await user.type(screen.getByLabelText('Name'), 'Alice');
    await user.click(screen.getByRole('button', { name: 'Send Message' }));

    // Button should be disabled while submitting
    expect(screen.getByRole('button', { name: /sending/i })).toBeDisabled();

    // After completion: button re-enables
    await screen.findByRole('alert');
    expect(screen.getByRole('button', { name: 'Send Message' })).toBeEnabled();
  });
});
```

---

## 10. Testing Routing and Navigation

```typescript
import { render, screen } from '@testing-library/react';
import { MemoryRouter, Routes, Route } from 'react-router-dom';
import userEvent from '@testing-library/user-event';

// Render with a specific initial URL
function renderWithRouter(ui: React.ReactElement, { initialEntries = ['/'] } = {}) {
  return render(
    <MemoryRouter initialEntries={initialEntries}>
      {ui}
    </MemoryRouter>
  );
}

test('navigates to product detail on click', async () => {
  const user = userEvent.setup();

  render(
    <MemoryRouter>
      <Routes>
        <Route path="/" element={<ProductList />} />
        <Route path="/products/:id" element={<ProductDetail />} />
      </Routes>
    </MemoryRouter>
  );

  // Should show list initially
  await screen.findByRole('list', { name: 'Products' });

  // Click a product
  await user.click(screen.getByRole('link', { name: 'Widget Pro' }));

  // Should navigate to product detail
  await screen.findByRole('heading', { name: 'Widget Pro' });
  expect(screen.queryByRole('list', { name: 'Products' })).not.toBeInTheDocument();
});

test('redirects unauthenticated users to login', () => {
  // Render a protected route when not authenticated
  renderWithRouter(<ProtectedRoute component={Dashboard} />, {
    initialEntries: ['/dashboard'],
  });

  // Should redirect to login with return path
  expect(screen.getByRole('heading', { name: 'Sign In' })).toBeInTheDocument();
  expect(screen.queryByRole('heading', { name: 'Dashboard' })).not.toBeInTheDocument();
});
```

---

## 11. Testing with Context and Providers

```typescript
// Testing a component that consumes context

// Option 1: Use the custom render wrapper (recommended)
import { render, screen } from '../test-utils'; // has providers built in

test('shows themed button', () => {
  render(<MyButton />); // ThemeProvider already wrapped
  expect(screen.getByRole('button')).toHaveClass('theme-dark');
});

// Option 2: Wrap manually for specific context values
import { ThemeContext } from '../contexts/ThemeContext';

test('renders correctly in light theme', () => {
  render(
    <ThemeContext.Provider value={{ theme: 'light', toggle: jest.fn() }}>
      <ThemedComponent />
    </ThemeContext.Provider>
  );
  expect(screen.getByTestId('theme-indicator')).toHaveTextContent('light');
});

test('renders correctly in dark theme', () => {
  render(
    <ThemeContext.Provider value={{ theme: 'dark', toggle: jest.fn() }}>
      <ThemedComponent />
    </ThemeContext.Provider>
  );
  expect(screen.getByTestId('theme-indicator')).toHaveTextContent('dark');
});

// Testing context mutations
test('toggles theme on button click', async () => {
  const user = userEvent.setup();
  const toggle = jest.fn();

  render(
    <ThemeContext.Provider value={{ theme: 'light', toggle }}>
      <ThemeToggleButton />
    </ThemeContext.Provider>
  );

  await user.click(screen.getByRole('button', { name: /toggle theme/i }));
  expect(toggle).toHaveBeenCalledTimes(1);
});
```

---

## 12. Testing Custom Hooks

Use `renderHook` from RTL to test hooks without creating a wrapper component.

```typescript
import { renderHook, act, waitFor } from '@testing-library/react';
import { useCounter } from './useCounter';
import { useProducts } from './useProducts';

// Testing a simple synchronous hook
test('useCounter increments and decrements', () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });
  expect(result.current.count).toBe(1);

  act(() => {
    result.current.decrement();
  });
  expect(result.current.count).toBe(0);

  act(() => {
    result.current.reset();
  });
  expect(result.current.count).toBe(0);
});

// Testing an async hook (with API call)
test('useProducts fetches and returns products', async () => {
  const { result } = renderHook(() => useProducts(), {
    wrapper: ({ children }) => (
      <QueryClientProvider client={new QueryClient()}>
        {children}
      </QueryClientProvider>
    ),
  });

  // Initially loading
  expect(result.current.isLoading).toBe(true);
  expect(result.current.products).toBeUndefined();

  // Wait for data
  await waitFor(() => expect(result.current.isLoading).toBe(false));

  expect(result.current.products).toHaveLength(2);
  expect(result.current.error).toBeNull();
});

// Testing hook with arguments
test('useDebounce delays value updates', async () => {
  jest.useFakeTimers();

  const { result, rerender } = renderHook(
    ({ value, delay }) => useDebounce(value, delay),
    { initialProps: { value: 'initial', delay: 500 } }
  );

  expect(result.current).toBe('initial');

  rerender({ value: 'updated', delay: 500 });
  expect(result.current).toBe('initial'); // not yet debounced

  act(() => jest.advanceTimersByTime(500));
  expect(result.current).toBe('updated'); // now updated

  jest.useRealTimers();
});
```

---

## 13. Snapshot Testing — When and How

Snapshot tests record component output and fail when it changes. Use them sparingly.

### When Snapshots Are Useful

```typescript
// ✅ Good: stable UI components, design system elements
test('Button renders correctly', () => {
  const { container } = render(
    <Button variant="primary" size="large">Click Me</Button>
  );
  expect(container.firstChild).toMatchSnapshot();
});

// ✅ Good: serialized data structures (not UI)
test('API response shape', () => {
  expect(formatUserData(rawUser)).toMatchSnapshot();
});
```

### When Snapshots Are NOT Useful

```typescript
// ❌ Bad: frequently changing components
test('Dashboard matches snapshot', () => {
  render(<Dashboard />);
  // Dashboard changes constantly — snapshot fails on every UI update
  // Developer blindly updates snapshot → defeats the purpose
});

// ❌ Bad: massive component trees
// 500-line snapshot diffs are useless for code review
```

### Targeted Snapshots

```typescript
// ✅ Snapshot specific parts, not entire pages
test('error state renders correctly', () => {
  render(<ProductList error="Failed to load" />);
  // Just snapshot the error component, not the entire page
  expect(screen.getByRole('alert')).toMatchSnapshot();
});
```

---

## 14. Good Practices

### ✅ Structure tests as Arrange-Act-Assert

```typescript
test('user can update their profile name', async () => {
  // ARRANGE
  const user = userEvent.setup();
  render(<ProfileForm initialName="Alice" />);

  // ACT
  await user.clear(screen.getByLabelText('Display Name'));
  await user.type(screen.getByLabelText('Display Name'), 'Alice Smith');
  await user.click(screen.getByRole('button', { name: 'Save' }));

  // ASSERT
  await screen.findByText('Profile updated');
  expect(screen.getByLabelText('Display Name')).toHaveValue('Alice Smith');
});
```

### ✅ One logical concept per test

```typescript
// ❌ Too much in one test
test("login form", async () => {
  // Tests validation AND happy path AND error state AND loading state
  // Fails in the middle → unclear what broke
});

// ✅ Focused tests
test("shows validation error for empty email");
test("shows validation error for invalid email format");
test("navigates to dashboard on successful login");
test("shows error message on invalid credentials");
test("disables submit button while logging in");
```

### ✅ Use descriptive test names as documentation

```typescript
describe("ProductCard", () => {
  describe("when product is in stock", () => {
    test('shows "Add to Cart" button');
    test("shows price in correct format");
    test("shows stock quantity");
  });

  describe("when product is out of stock", () => {
    test('shows "Out of Stock" indicator instead of button');
    test('shows "Notify Me" option');
    test("disables Add to Cart button");
  });
});
```

### ✅ Avoid hardcoding text strings — use constants

```typescript
const TEXTS = {
  SUBMIT_BUTTON: "Place Order",
  SUCCESS_MESSAGE: "Order confirmed",
  ERROR_MESSAGE: "Payment failed. Please try again.",
} as const;

test("shows success after payment", async () => {
  await user.click(screen.getByRole("button", { name: TEXTS.SUBMIT_BUTTON }));
  await screen.findByText(TEXTS.SUCCESS_MESSAGE);
});
```

---

## 15. Bad Practices

### ❌ Testing implementation details

```typescript
// ❌ Tests internal state — breaks on refactoring
test('sets isSubmitting to true on submit', () => {
  const { result } = renderHook(() => useForm());
  expect(result.current.isSubmitting).toBe(false);
  act(() => result.current.submit());
  expect(result.current.isSubmitting).toBe(true);
});

// ✅ Test what the user sees: the button is disabled
test('submit button is disabled while submitting', async () => {
  render(<MyForm />);
  await user.click(screen.getByRole('button', { name: 'Submit' }));
  expect(screen.getByRole('button', { name: /submitting/i })).toBeDisabled();
});
```

### ❌ Using `waitForTimeout` or `setTimeout` in tests

```typescript
// ❌ Slow and non-deterministic
test('shows loading', async () => {
  render(<AsyncComponent />);
  await new Promise(r => setTimeout(r, 1000)); // flaky!
  expect(screen.getByText('Loaded')).toBeInTheDocument();
});

// ✅ Wait for the actual condition
test('shows content after loading', async () => {
  render(<AsyncComponent />);
  await screen.findByText('Loaded'); // retries until present
});
```

### ❌ Not cleaning up between tests

```typescript
// ❌ Server handlers not reset — bleed between tests
test('test A - uses default handler');
test('test B - overrides handler'); // server.use(...)
test('test C - should use default but gets test B's override!');

// ✅ Reset in afterEach (already in jest.setup.js with MSW)
afterEach(() => server.resetHandlers());
```

### ❌ Mocking too deeply

```typescript
// ❌ Mocking at too low a level
jest.mock("../utils/formatPrice");
jest.mock("../services/productService");
jest.mock("../hooks/useCart");
// ... mocking 8 more modules

// These mocks make the test pass even if the real modules are broken
// You're testing a mock, not your code

// ✅ Mock only external boundaries (HTTP, Date, Math.random)
// Let everything else be real
```

---

## 16. Common Mistakes

### Mistake 1 — Not wrapping state updates in `act()`

```typescript
// Error: "Warning: An update to... inside a test was not wrapped in act(...)"
test('counter updates', () => {
  render(<Counter />);
  fireEvent.click(screen.getByRole('button', { name: 'Increment' }));
  // State update happens but test doesn't wait for React to process it
  expect(screen.getByText('1')).toBeInTheDocument(); // may fail
});

// ✅ Use userEvent (wraps in act automatically)
test('counter updates', async () => {
  const user = userEvent.setup();
  render(<Counter />);
  await user.click(screen.getByRole('button', { name: 'Increment' }));
  expect(screen.getByText('1')).toBeInTheDocument(); // reliable
});
```

### Mistake 2 — Forgetting `await` with `findBy*`

```typescript
// ❌ Returns a Promise, not the element
const element = screen.findByText("Loading complete"); // Promise!
expect(element).toBeInTheDocument(); // ALWAYS passes (Promise is truthy)

// ✅ Await the Promise
const element = await screen.findByText("Loading complete");
expect(element).toBeInTheDocument();
```

### Mistake 3 — Testing with `getBy*` when element loads asynchronously

```typescript
// ❌ Element not yet present — throws immediately
expect(screen.getByText("Product loaded")).toBeInTheDocument();

// ✅ Use findBy* for async elements (retries)
await screen.findByText("Product loaded");
expect(screen.getByText("Product loaded")).toBeInTheDocument();
// Or just:
await expect(screen.findByText("Product loaded")).resolves.toBeInTheDocument();
```

### Mistake 4 — Not resetting mocks between tests

```typescript
const mockOnSubmit = jest.fn();

test('first test calls onSubmit once', async () => {
  render(<Form onSubmit={mockOnSubmit} />);
  await user.click(submitButton);
  expect(mockOnSubmit).toHaveBeenCalledTimes(1);
});

test('second test calls onSubmit once', async () => {
  render(<Form onSubmit={mockOnSubmit} />);
  await user.click(submitButton);
  expect(mockOnSubmit).toHaveBeenCalledTimes(1); // FAILS: called twice total!
});

// ✅ Clear mocks between tests
beforeEach(() => {
  jest.clearAllMocks(); // clears call counts
});
// Or: jest.resetAllMocks() (also resets return values)
// Or: jest.restoreAllMocks() (also restores spied implementations)
```

---

## 17. Interview-Level Explanation

> **"What is integration testing? How do you approach it for frontend components?"**

**Strong answer:**

> "Integration tests verify that multiple units work correctly together. For frontend components, this means rendering a real component tree and asserting that it behaves correctly when a user interacts with it — not testing the component's internal state or implementation details.
>
> The core philosophy, from the React Testing Library approach, is to test your application the way users use it. Instead of checking that a component's `isSubmitting` state becomes `true`, you check that the submit button is disabled and shows 'Submitting...' — which is what the user actually experiences. Tests written this way survive internal refactoring because they're testing the contract (behavior) rather than the mechanism (implementation).
>
> For tooling, I use React Testing Library with `@testing-library/user-event` for interactions. RTL renders into a real jsdom DOM and provides queries that mirror how users find elements — by role, label, text, and so on. `userEvent` simulates realistic sequences of events (focus, keydown, input, etc.) rather than single events, which catches more real-world bugs.
>
> For HTTP mocking, MSW (Mock Service Worker) is the right tool. It intercepts requests at the network level, so it works with any HTTP client and tests that your component sends the correct request shape. You define handlers once and override them per-test when you need specific scenarios — 500 errors, empty responses, slow responses.
>
> A test I consider well-written for a form: it renders the form, fills in fields using `userEvent.type`, clicks submit, verifies the correct data was sent to the mocked API, and verifies the success UI appears. If a server error occurs, it verifies the form preserves the user's input and shows an actionable error. That's a complete test of the user's experience, written in maybe 30 lines, that runs in 100ms and will catch most real bugs."

---

## 18. Exercises

### Exercise 1 — Write a complete component test suite

Write integration tests for this `SearchBar` component:

- Renders a text input and a search button
- Calls `onSearch(query)` when form is submitted
- Shows a clear button when there is input, clears on click
- Shows loading indicator while `isLoading` prop is true
- Disables input and button while loading

```typescript
interface SearchBarProps {
  onSearch: (query: string) => void;
  isLoading?: boolean;
}
```

<details>
<summary>Solution</summary>

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SearchBar } from './SearchBar';

describe('SearchBar', () => {
  const onSearch = jest.fn();
  const user = userEvent.setup();

  beforeEach(() => jest.clearAllMocks());

  test('renders input and search button', () => {
    render(<SearchBar onSearch={onSearch} />);

    expect(screen.getByRole('searchbox')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'Search' })).toBeInTheDocument();
  });

  test('calls onSearch with input value on form submit', async () => {
    render(<SearchBar onSearch={onSearch} />);

    await user.type(screen.getByRole('searchbox'), 'headphones');
    await user.click(screen.getByRole('button', { name: 'Search' }));

    expect(onSearch).toHaveBeenCalledTimes(1);
    expect(onSearch).toHaveBeenCalledWith('headphones');
  });

  test('calls onSearch on Enter key press', async () => {
    render(<SearchBar onSearch={onSearch} />);

    await user.type(screen.getByRole('searchbox'), 'headphones{Enter}');

    expect(onSearch).toHaveBeenCalledWith('headphones');
  });

  test('shows clear button when there is input', async () => {
    render(<SearchBar onSearch={onSearch} />);

    expect(screen.queryByRole('button', { name: 'Clear search' })).not.toBeInTheDocument();

    await user.type(screen.getByRole('searchbox'), 'test');

    expect(screen.getByRole('button', { name: 'Clear search' })).toBeInTheDocument();
  });

  test('clears input when clear button is clicked', async () => {
    render(<SearchBar onSearch={onSearch} />);

    await user.type(screen.getByRole('searchbox'), 'test query');
    await user.click(screen.getByRole('button', { name: 'Clear search' }));

    expect(screen.getByRole('searchbox')).toHaveValue('');
    expect(screen.queryByRole('button', { name: 'Clear search' })).not.toBeInTheDocument();
  });

  test('shows loading indicator while isLoading is true', () => {
    render(<SearchBar onSearch={onSearch} isLoading />);

    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });

  test('disables input and button while loading', () => {
    render(<SearchBar onSearch={onSearch} isLoading />);

    expect(screen.getByRole('searchbox')).toBeDisabled();
    expect(screen.getByRole('button', { name: 'Search' })).toBeDisabled();
  });

  test('does not call onSearch when submitted with empty input', async () => {
    render(<SearchBar onSearch={onSearch} />);

    await user.click(screen.getByRole('button', { name: 'Search' }));

    expect(onSearch).not.toHaveBeenCalled();
  });
});
```

</details>

---

### Exercise 2 — Fix the test

This test is broken. Identify all issues and fix them:

```typescript
test('product page loads and shows details', () => {
  render(<ProductPage id="1" />);

  // Check title is rendered
  expect(screen.getByText('Widget Pro')).toBeInTheDocument();

  // Check price
  expect(screen.getByText('$29.99')).toBeInTheDocument();

  // Check add to cart button
  const button = screen.findByRole('button', { name: 'Add to Cart' });
  expect(button).toBeEnabled();
});
```

<details>
<summary>Answer</summary>

```typescript
// Issues:
// 1. test() should be async — the component fetches data asynchronously
// 2. getByText() will fail if product data hasn't loaded yet
//    → use findByText() which awaits the element
// 3. screen.findByRole() returns a Promise — missing await
//    → the expect(button).toBeEnabled() is checking the Promise object (always truthy)

// Fixed:
test('product page loads and shows details', async () => {
  render(<ProductPage id="1" />);

  // Wait for async data to load
  const title = await screen.findByText('Widget Pro');
  expect(title).toBeInTheDocument();

  // Now safe to use getBy (data is loaded)
  expect(screen.getByText('$29.99')).toBeInTheDocument();

  // Await the findBy call
  const button = await screen.findByRole('button', { name: 'Add to Cart' });
  expect(button).toBeEnabled();
});
```

</details>

---

## 🔗 Related Topics

- [`testing/01-unit-testing.md`](./01-unit-testing.md) — Unit testing fundamentals
- [`testing/03-e2e-testing.md`](./03-e2e-testing.md) — End-to-end testing with Playwright
- [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md) — Async patterns tested here
- [`debugging/01-browser-devtools.md`](../debugging/01-browser-devtools.md) — Debugging failing tests

---

<div align="center">

**`testing/` section:** [`01-unit-testing.md`](./01-unit-testing.md) · **`02-integration-testing.md`** ✓ · [`03-e2e-testing.md`](./03-e2e-testing.md) ✓

</div>
