# 03 — End-to-End Testing

> **"Unit tests tell you your functions work. Integration tests tell you your modules work together. E2E tests tell you your users can actually do the thing they came to do. All three matter — but only E2E tests the system as a whole."**

End-to-end (E2E) tests automate a real browser to verify complete user journeys — from clicking a button to confirming the right data appears in the database. They are the highest-confidence tests in the testing pyramid and the last line of defense before production. This document covers Playwright (the modern standard), E2E architecture, page objects, CI integration, flakiness elimination, and the strategic thinking behind what to E2E test and what not to.

---

## 📚 Table of Contents

1. [What E2E Testing Is and Isn't](#1-what-e2e-testing-is-and-isnt)
2. [The Testing Trophy — Where E2E Fits](#2-the-testing-trophy--where-e2e-fits)
3. [Playwright — The Modern Standard](#3-playwright--the-modern-standard)
4. [Setup and Configuration](#4-setup-and-configuration)
5. [Writing Your First E2E Test](#5-writing-your-first-e2e-test)
6. [Selectors — What to Use and Why](#6-selectors--what-to-use-and-why)
7. [Page Object Model](#7-page-object-model)
8. [Network Interception and Mocking](#8-network-interception-and-mocking)
9. [Authentication Patterns](#9-authentication-patterns)
10. [Waiting Strategies — Eliminating Flakiness](#10-waiting-strategies--eliminating-flakiness)
11. [Visual Regression Testing](#11-visual-regression-testing)
12. [Accessibility Testing in E2E](#12-accessibility-testing-in-e2e)
13. [CI/CD Integration](#13-cicd-integration)
14. [Test Data Management](#14-test-data-management)
15. [Debugging Failing Tests](#15-debugging-failing-tests)
16. [Good Practices](#16-good-practices)
17. [Bad Practices](#17-bad-practices)
18. [Common Mistakes](#18-common-mistakes)
19. [Interview-Level Explanation](#19-interview-level-explanation)
20. [Exercises](#20-exercises)

---

## 1. What E2E Testing Is and Isn't

### What E2E Testing Is

E2E tests automate a real browser against your actual application (or a close replica of it) and verify that complete user workflows produce the expected outcomes.

```
E2E test for "user purchases a product":
  1. Browser opens https://staging.shop.com
  2. User searches for "headphones"
  3. Clicks first result
  4. Clicks "Add to Cart"
  5. Navigates to checkout
  6. Fills in shipping + payment
  7. Clicks "Place Order"
  8. Asserts: order confirmation page shown
  9. Asserts: email received (optional)
  10. Asserts: order appears in admin dashboard (optional)

This test exercises: frontend, API, database, email service
If any layer fails: the test fails
```

### What E2E Testing Is NOT

```
❌ A replacement for unit or integration tests
   (E2E tests are slow and expensive — test edge cases lower in the pyramid)

❌ A complete specification for every code path
   (Test user journeys, not every function permutation)

❌ Always real infrastructure
   (Network calls can be mocked for reliability)

❌ Run on every commit
   (Usually: key paths on PR, full suite on merge to main)
```

### The E2E Test Pyramid Position

```
           /\
          /  \
         / E2E\   ← Few tests, high confidence, slow, expensive
        /──────\
       /  Integ \  ← More tests, good confidence, moderate speed
      /──────────\
     /   Unit     \ ← Many tests, specific confidence, fast, cheap
    /______________\
```

E2E tests sit at the top: expensive to write, slow to run, but providing unmatched confidence that real users can complete real workflows.

---

## 2. The Testing Trophy — Where E2E Fits

Kent C. Dodds' Testing Trophy is a useful refinement of the pyramid:

```
        🏆
       /  \
      / E2E \       ← Smoke tests + critical paths only
     /────────\
    / Integration\  ← Most tests live here (React Testing Library style)
   /──────────────\
  /     Unit       \ ← Pure functions, utilities, algorithms
 /──────────────────\
/    Static Analysis \ ← TypeScript, ESLint — catches bugs for free
\____________________/
```

**The E2E sweet spot:** smoke tests that verify the app loads and core journeys work, plus critical paths (checkout, auth, key feature) that have the highest business impact if broken.

---

## 3. Playwright — The Modern Standard

Playwright (by Microsoft) has become the dominant E2E framework as of 2023-2024:

```
Playwright vs alternatives:
  Playwright:
    ✓ Chromium, Firefox, WebKit (Safari) — one API
    ✓ Auto-waiting built in — no manual waits
    ✓ Network interception
    ✓ Multiple pages/contexts in one test
    ✓ Trace viewer (debugging)
    ✓ Component testing mode
    ✓ TypeScript first-class
    ✓ Parallelism by default

  Cypress:
    ✓ Excellent developer experience
    ✓ Time-travel debugging
    ✗ Chromium-only (Firefox beta)
    ✗ No multi-tab support
    ✗ Slower than Playwright

  Selenium/WebDriver:
    ✓ Oldest — widest browser support
    ✗ Verbose API, flaky by default
    ✗ No auto-waiting
    ✗ Steep setup cost
```

### Playwright's Core Model

```javascript
// Three core objects:

// 1. Browser — a browser instance (Chromium, Firefox, WebKit)
const browser = await chromium.launch();

// 2. BrowserContext — an isolated browser session (like an incognito window)
// Separate cookies, localStorage, auth state per context
const context = await browser.newContext();

// 3. Page — a single browser tab
const page = await context.newPage();

// Playwright manages all three automatically in tests
// You usually just get a `page` from the `test` fixture
```

---

## 4. Setup and Configuration

### Installation

```bash
npm init playwright@latest
# Creates: playwright.config.ts, tests/example.spec.ts
# Installs: @playwright/test + browser binaries

# Install browsers manually:
npx playwright install
npx playwright install --with-deps # includes system dependencies (for CI)
```

### `playwright.config.ts`

```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  // Test directory
  testDir: "./tests",

  // Run tests in parallel
  fullyParallel: true,

  // Fail the CI build if any test.only is left
  forbidOnly: !!process.env.CI,

  // Retry failed tests on CI only
  retries: process.env.CI ? 2 : 0,

  // Workers (parallel test files)
  workers: process.env.CI ? 2 : undefined,

  // Reporter
  reporter: [
    ["html"], // local HTML report
    ["github"], // GitHub Actions annotations (CI)
    ["json", { outputFile: "test-results/results.json" }],
  ],

  // Shared settings for all tests
  use: {
    // Base URL — all page.goto('/path') calls use this
    baseURL: process.env.BASE_URL ?? "http://localhost:3000",

    // Record traces on first retry of failed test
    trace: "on-first-retry",

    // Screenshot on failure
    screenshot: "only-on-failure",

    // Video on failure
    video: "on-first-retry",

    // Default timeout for actions (click, fill, etc.)
    actionTimeout: 10_000,

    // Navigation timeout
    navigationTimeout: 30_000,
  },

  // Test projects (browser configurations)
  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
    {
      name: "firefox",
      use: { ...devices["Desktop Firefox"] },
    },
    {
      name: "webkit",
      use: { ...devices["Desktop Safari"] },
    },
    // Mobile
    {
      name: "mobile-chrome",
      use: { ...devices["Pixel 7"] },
    },
    {
      name: "mobile-safari",
      use: { ...devices["iPhone 14"] },
    },
  ],

  // Start local dev server before tests
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

---

## 5. Writing Your First E2E Test

```typescript
// tests/auth.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Authentication", () => {
  test("user can log in with valid credentials", async ({ page }) => {
    // Navigate
    await page.goto("/login");

    // Interact
    await page.getByLabel("Email").fill("user@example.com");
    await page.getByLabel("Password").fill("correctpassword");
    await page.getByRole("button", { name: "Sign In" }).click();

    // Assert
    await expect(page).toHaveURL("/dashboard");
    await expect(page.getByText("Welcome back")).toBeVisible();
    await expect(page.getByRole("navigation")).toContainText("My Account");
  });

  test("shows error with invalid credentials", async ({ page }) => {
    await page.goto("/login");

    await page.getByLabel("Email").fill("user@example.com");
    await page.getByLabel("Password").fill("wrongpassword");
    await page.getByRole("button", { name: "Sign In" }).click();

    // Assert error state — stays on login page
    await expect(page).toHaveURL("/login");
    await expect(page.getByRole("alert")).toContainText(
      "Invalid email or password",
    );
    await expect(page.getByLabel("Password")).toHaveValue(""); // cleared on error
  });

  test("redirects to intended page after login", async ({ page }) => {
    // Navigate directly to protected page
    await page.goto("/settings/billing");

    // Should redirect to login
    await expect(page).toHaveURL(/\/login\?redirect=/);

    // Log in
    await page.getByLabel("Email").fill("user@example.com");
    await page.getByLabel("Password").fill("correctpassword");
    await page.getByRole("button", { name: "Sign In" }).click();

    // Should redirect to original destination
    await expect(page).toHaveURL("/settings/billing");
  });
});
```

### Assertions Reference

```typescript
// URL
await expect(page).toHaveURL("/dashboard");
await expect(page).toHaveURL(/dashboard/); // regex

// Title
await expect(page).toHaveTitle("My App");

// Element visibility
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();

// Text content
await expect(locator).toHaveText("Exact match");
await expect(locator).toContainText("partial match");
await expect(locator).toHaveText(/regex/);

// Attribute / value
await expect(locator).toHaveAttribute("href", "/home");
await expect(locator).toHaveValue("input value"); // for inputs
await expect(locator).toBeChecked(); // checkbox/radio

// Count
await expect(locator).toHaveCount(5);

// CSS
await expect(locator).toHaveClass("active");
await expect(locator).toHaveCSS("color", "rgb(0, 0, 0)");

// Element state
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();
await expect(locator).toBeFocused();

// Snapshot
await expect(page).toHaveScreenshot("my-page.png");
```

---

## 6. Selectors — What to Use and Why

Playwright supports multiple selector strategies. The hierarchy, from most to least preferred:

### Priority 1 — User-Facing Attributes (Best)

```typescript
// By role — most semantic, mirrors how users experience the page
page.getByRole("button", { name: "Submit" });
page.getByRole("link", { name: "Home" });
page.getByRole("textbox", { name: "Search" });
page.getByRole("heading", { name: "Dashboard", level: 1 });
page.getByRole("checkbox", { name: "Remember me" });
page.getByRole("tab", { name: "Settings" });
page.getByRole("dialog", { name: "Confirm Delete" });

// By label — for form inputs
page.getByLabel("Email Address");
page.getByLabel("Password");

// By placeholder text
page.getByPlaceholder("Search products...");

// By text content
page.getByText("Welcome back, Alice");
page.getByText(/welcome back/i); // case-insensitive regex

// By alt text (images)
page.getByAltText("Company logo");

// By title attribute
page.getByTitle("Close dialog");
```

### Priority 2 — Test IDs (When Roles Aren't Enough)

```typescript
// By test ID — explicit marker, not exposed to users
page.getByTestId("product-card-42");
page.getByTestId("submit-button");

// HTML:
// <button data-testid="submit-button">Submit</button>

// Configure the attribute name (default: data-testid):
// playwright.config.ts: use: { testIdAttribute: 'data-test' }
```

### Priority 3 — CSS Selectors (Last Resort)

```typescript
// CSS: use when semantic selectors aren't sufficient
page.locator(".product-card:first-child");
page.locator('[data-status="active"]');

// Avoid: brittle, tied to implementation
page.locator("div > ul > li:nth-child(3) > span.title"); // ❌ fragile
```

### Chaining and Filtering

```typescript
// Within a container
const productCard = page.locator(".product-card").first();
await productCard.getByRole("button", { name: "Add to Cart" }).click();

// Filter by text
const activeItems = page.getByRole("listitem").filter({ hasText: "Active" });

// Filter by another locator
const checkedItems = page.getByRole("listitem").filter({
  has: page.getByRole("checkbox", { checked: true }),
});

// nth element
await page.getByRole("row").nth(2).click(); // 0-indexed

// Last element
await page.getByRole("row").last().click();
```

---

## 7. Page Object Model

The Page Object Model (POM) encapsulates page interactions into reusable classes. Tests describe the user journey; POMs describe how to interact with a page.

### Basic Page Object

```typescript
// tests/pages/LoginPage.ts
import { Page, Locator, expect } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorAlert: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("Email");
    this.passwordInput = page.getByLabel("Password");
    this.submitButton = page.getByRole("button", { name: "Sign In" });
    this.errorAlert = page.getByRole("alert");
  }

  async goto() {
    await this.page.goto("/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorAlert).toContainText(message);
  }
}
```

```typescript
// tests/pages/DashboardPage.ts
import { Page, Locator, expect } from "@playwright/test";

export class DashboardPage {
  readonly page: Page;
  readonly heading: Locator;
  readonly userMenu: Locator;
  readonly logoutButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.heading = page.getByRole("heading", { level: 1 });
    this.userMenu = page.getByRole("button", { name: /account/i });
    this.logoutButton = page.getByRole("menuitem", { name: "Sign Out" });
  }

  async goto() {
    await this.page.goto("/dashboard");
  }

  async logout() {
    await this.userMenu.click();
    await this.logoutButton.click();
  }

  async expectLoaded() {
    await expect(this.page).toHaveURL("/dashboard");
    await expect(this.heading).toBeVisible();
  }
}
```

```typescript
// tests/auth.spec.ts — clean test using POMs
import { test, expect } from "@playwright/test";
import { LoginPage } from "./pages/LoginPage";
import { DashboardPage } from "./pages/DashboardPage";

test("successful login journey", async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login("user@example.com", "password123");
  await dashboardPage.expectLoaded();

  await dashboardPage.logout();
  await expect(page).toHaveURL("/login");
});
```

### Component Objects (Sub-Page Objects)

```typescript
// tests/components/NavBar.ts — reusable component across pages
import { Page, Locator, expect } from "@playwright/test";

export class NavBar {
  private readonly nav: Locator;

  constructor(page: Page) {
    this.nav = page.getByRole("navigation", { name: "Main" });
  }

  link(name: string) {
    return this.nav.getByRole("link", { name });
  }

  async expectActiveLink(name: string) {
    await expect(this.link(name)).toHaveAttribute("aria-current", "page");
  }

  async navigate(name: string) {
    await this.link(name).click();
  }
}
```

---

## 8. Network Interception and Mocking

Playwright can intercept and modify network requests — enabling tests that run without real backends, or tests of specific error conditions.

### Intercepting API Calls

```typescript
test("shows error message when API fails", async ({ page }) => {
  // Intercept the products API and return an error
  await page.route("**/api/products", async (route) => {
    await route.fulfill({
      status: 500,
      contentType: "application/json",
      body: JSON.stringify({ error: "Internal Server Error" }),
    });
  });

  await page.goto("/products");

  await expect(page.getByRole("alert")).toContainText(
    "Failed to load products",
  );
  await expect(page.getByRole("button", { name: "Try Again" })).toBeVisible();
});
```

### Mocking Successful Responses

```typescript
test("renders product list from API", async ({ page }) => {
  const mockProducts = [
    { id: 1, name: "Widget Pro", price: 29.99, inStock: true },
    { id: 2, name: "Gadget Max", price: 49.99, inStock: false },
  ];

  await page.route("**/api/products", async (route) => {
    await route.fulfill({
      status: 200,
      contentType: "application/json",
      body: JSON.stringify(mockProducts),
    });
  });

  await page.goto("/products");

  await expect(page.getByTestId("product-list")).toBeVisible();
  await expect(page.getByRole("listitem")).toHaveCount(2);
  await expect(page.getByText("Widget Pro")).toBeVisible();
  await expect(page.getByText("Gadget Max")).toBeVisible();
  await expect(page.getByText("Out of Stock")).toBeVisible();
});
```

### Modify Requests (Pass-Through with Changes)

```typescript
// Add auth header to all API requests
test("uses auth token in API calls", async ({ page }) => {
  await page.route("**/api/**", async (route) => {
    const headers = {
      ...route.request().headers(),
      Authorization: "Bearer test-token",
    };
    await route.continue({ headers });
  });

  // Verify the token is sent (via request inspection)
  const [request] = await Promise.all([
    page.waitForRequest("**/api/profile"),
    page.goto("/profile"),
  ]);

  expect(request.headers()["authorization"]).toBe("Bearer test-token");
});
```

### Network Waiting Patterns

```typescript
// Wait for a specific API call and inspect it
test("posts form data correctly", async ({ page }) => {
  await page.goto("/contact");

  await page.getByLabel("Name").fill("Alice");
  await page.getByLabel("Message").fill("Hello!");

  // Set up promise BEFORE the action that triggers the request
  const [request] = await Promise.all([
    page.waitForRequest("**/api/contact"),
    page.getByRole("button", { name: "Send" }).click(),
  ]);

  const body = JSON.parse(request.postData() ?? "{}");
  expect(body.name).toBe("Alice");
  expect(body.message).toBe("Hello!");
});

// Wait for response
const [response] = await Promise.all([
  page.waitForResponse("**/api/contact"),
  page.getByRole("button", { name: "Send" }).click(),
]);
expect(response.status()).toBe(201);
```

---

## 9. Authentication Patterns

Authentication is required for most E2E tests. Logging in through the UI before every test is slow and fragile.

### Pattern 1 — Store Auth State (Recommended)

```typescript
// tests/auth.setup.ts — run once before all tests
import { test as setup, expect } from "@playwright/test";
import path from "path";

const authFile = path.join(__dirname, "../playwright/.auth/user.json");

setup("authenticate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill(process.env.TEST_EMAIL!);
  await page.getByLabel("Password").fill(process.env.TEST_PASSWORD!);
  await page.getByRole("button", { name: "Sign In" }).click();
  await expect(page).toHaveURL("/dashboard");

  // Save authentication state (cookies + localStorage)
  await page.context().storageState({ path: authFile });
});
```

```typescript
// playwright.config.ts — configure projects
export default defineConfig({
  projects: [
    // Setup project: runs once, saves auth state
    {
      name: "setup",
      testMatch: /.*\.setup\.ts/,
    },
    // Main tests: use saved auth state
    {
      name: "authenticated",
      use: {
        storageState: "playwright/.auth/user.json",
      },
      dependencies: ["setup"],
    },
    // Tests that don't need auth
    {
      name: "unauthenticated",
      use: {
        storageState: undefined,
      },
    },
  ],
});
```

### Pattern 2 — API-Based Authentication (Fastest)

```typescript
// Skip the UI entirely — call the auth API directly
import { test, expect } from "@playwright/test";
import { request } from "@playwright/test";

test.beforeAll(async ({ playwright }) => {
  const apiContext = await playwright.request.newContext({
    baseURL: "https://api.example.com",
  });

  // Get token via API
  const response = await apiContext.post("/auth/login", {
    data: {
      email: process.env.TEST_EMAIL,
      password: process.env.TEST_PASSWORD,
    },
  });

  const { token } = await response.json();

  // Store token for all tests
  process.env.AUTH_TOKEN = token;
});

test("accesses protected page with token", async ({ page }) => {
  // Set auth cookie/token before navigation
  await page.context().addCookies([
    {
      name: "auth_token",
      value: process.env.AUTH_TOKEN!,
      domain: "localhost",
      path: "/",
    },
  ]);

  await page.goto("/dashboard");
  await expect(page.getByRole("heading", { level: 1 })).toBeVisible();
});
```

### Pattern 3 — Test-Only Auth Bypass

```typescript
// Application provides a test-only endpoint that issues auth without password
// ONLY available in test/staging environments (never production)

test("uses test auth endpoint", async ({ page }) => {
  if (process.env.NODE_ENV !== "test") {
    test.skip(true, "Test auth endpoint only available in test env");
  }

  // Call test-only endpoint to get a session
  await page.goto(
    `/api/test/auth?userId=${TEST_USER_ID}&secret=${process.env.TEST_SECRET}`,
  );

  // Now authenticated — navigate to protected page
  await page.goto("/dashboard");
  await expect(page).toHaveURL("/dashboard");
});
```

---

## 10. Waiting Strategies — Eliminating Flakiness

Flaky tests are the enemy of E2E confidence. Most flakiness comes from incorrect waiting.

### Playwright's Auto-Waiting

Playwright automatically waits for elements to be:

- Attached to DOM
- Visible
- Stable (not animating)
- Receiving events (not covered by another element)
- Enabled

```typescript
// These automatically wait for the element to be ready:
await page.getByRole("button", { name: "Submit" }).click();
await page.getByLabel("Email").fill("user@example.com");
await expect(locator).toBeVisible();
// No manual waits needed for standard interactions
```

### When Auto-Wait Isn't Enough

```typescript
// ❌ Avoid: fixed time waits — fragile and slow
await page.waitForTimeout(2000); // never do this

// ✅ Wait for specific condition
await page.waitForURL("/dashboard");
await page.waitForSelector(".product-list");
await page.waitForLoadState("networkidle"); // use sparingly

// ✅ Wait for element state
await expect(page.getByRole("progressbar")).toBeHidden();
await expect(page.getByRole("status")).toContainText("Saved");

// ✅ Wait for network request to complete
await page.waitForResponse("**/api/products");

// ✅ Custom condition
await page.waitForFunction(() => {
  const items = document.querySelectorAll(".item");
  return items.length > 0;
});
```

### Handling Async UI Updates

```typescript
// Pattern: action → wait for indicator → assert result
test("saves settings successfully", async ({ page }) => {
  await page.goto("/settings");

  await page.getByLabel("Display Name").fill("Alice Smith");
  await page.getByRole("button", { name: "Save Changes" }).click();

  // Wait for save to complete — look for success indicator
  await expect(page.getByRole("status")).toContainText("Settings saved");
  // OR wait for loading spinner to disappear
  await expect(page.getByRole("progressbar")).toBeHidden();

  // Now assert the persisted state
  await page.reload();
  await expect(page.getByLabel("Display Name")).toHaveValue("Alice Smith");
});
```

### `expect` Retry Behavior

Playwright's `expect` retries assertions automatically until the timeout:

```typescript
// This retries up to actionTimeout (default 5000ms):
await expect(page.getByText("Order confirmed")).toBeVisible();
// Not visible at first? Playwright keeps retrying until visible or timeout

// Increase timeout for slow operations:
await expect(page.getByText("Report generated")).toBeVisible({
  timeout: 30_000, // 30 seconds for slow reports
});

// Soft assertions: collect failures instead of stopping
await expect.soft(page.getByTestId("item-count")).toHaveText("5");
await expect.soft(page.getByTestId("total-price")).toHaveText("$49.95");
// Both assertions are checked — test reports all failures at once
```

---

## 11. Visual Regression Testing

Visual regression testing detects unintended UI changes by comparing screenshots.

### Playwright Screenshot Testing

```typescript
test("product page matches design", async ({ page }) => {
  await page.goto("/products/widget-pro");

  // Full page screenshot
  await expect(page).toHaveScreenshot("product-page.png");

  // Specific element screenshot
  await expect(page.getByTestId("product-hero")).toHaveScreenshot("hero.png");
});
```

### Updating Baselines

```bash
# First run: creates baseline screenshots
npx playwright test --update-snapshots

# Subsequent runs: compares against baseline
npx playwright test
# Failures: screenshot differs from baseline → shows diff image
```

### Screenshot Options

```typescript
await expect(page).toHaveScreenshot("page.png", {
  // Acceptable pixel difference
  maxDiffPixelRatio: 0.02, // 2% of pixels may differ

  // Clip to specific area
  clip: { x: 0, y: 0, width: 1280, height: 720 },

  // Mask dynamic content (timestamps, randomness)
  mask: [page.getByTestId("timestamp"), page.getByTestId("random-content")],

  // Animations: disable for stable screenshots
  animations: "disabled",
});
```

### When Visual Testing Makes Sense

```
Good candidates for visual regression:
  ✓ Design system components (buttons, forms, cards)
  ✓ Data visualizations (charts, graphs)
  ✓ Critical marketing pages
  ✓ Email templates

Poor candidates:
  ✗ Pages with dynamic content (timestamps, user data)
  ✗ Pages that change frequently
  ✗ Every single page (too many false positives)
```

---

## 12. Accessibility Testing in E2E

E2E tests are an excellent place to run automated accessibility checks.

### axe Integration

```typescript
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test("homepage has no accessibility violations", async ({ page }) => {
  await page.goto("/");

  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa", "wcag21aa"])
    .analyze();

  // Report violations (don't fail yet — review first run)
  if (results.violations.length > 0) {
    console.log("Accessibility violations:");
    results.violations.forEach((violation) => {
      console.log(
        `- [${violation.impact}] ${violation.id}: ${violation.description}`,
      );
      violation.nodes.forEach((node) => {
        console.log(`  Target: ${node.target}`);
      });
    });
  }

  expect(results.violations).toHaveLength(0);
});

// Scoped to a component
test("login form is accessible", async ({ page }) => {
  await page.goto("/login");

  const results = await new AxeBuilder({ page })
    .include("#login-form")
    .analyze();

  expect(results.violations).toHaveLength(0);
});

// Exclude known violations (with a tracking issue)
test("dashboard accessibility", async ({ page }) => {
  await page.goto("/dashboard");

  const results = await new AxeBuilder({ page })
    .exclude("#legacy-widget") // tracked in ISSUE-123
    .analyze();

  expect(results.violations).toHaveLength(0);
});
```

### Keyboard Navigation Testing

```typescript
test("modal is keyboard accessible", async ({ page }) => {
  await page.goto("/products");

  // Open modal via keyboard
  await page.getByRole("button", { name: "View Details" }).press("Enter");

  // Focus should be trapped inside modal
  const modal = page.getByRole("dialog");
  await expect(modal).toBeFocused();

  // Tab through interactive elements
  await page.keyboard.press("Tab");
  await expect(page.getByRole("button", { name: "Add to Cart" })).toBeFocused();

  await page.keyboard.press("Tab");
  await expect(page.getByRole("button", { name: "Close" })).toBeFocused();

  // Tab wraps back to first focusable element (focus trap)
  await page.keyboard.press("Tab");
  await expect(page.getByRole("button", { name: "Add to Cart" })).toBeFocused();

  // Escape closes modal
  await page.keyboard.press("Escape");
  await expect(modal).toBeHidden();
  // Focus returns to trigger button
  await expect(
    page.getByRole("button", { name: "View Details" }),
  ).toBeFocused();
});
```

---

## 13. CI/CD Integration

### GitHub Actions Configuration

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        # Shard tests across 4 parallel runners
        shardIndex: [1, 2, 3, 4]
        shardTotal: [4]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run E2E tests (shard ${{ matrix.shardIndex }}/${{ matrix.shardTotal }})
        run: |
          npx playwright test \
            --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          TEST_EMAIL: ${{ secrets.TEST_EMAIL }}
          TEST_PASSWORD: ${{ secrets.TEST_PASSWORD }}

      - name: Upload test artifacts
        uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report-${{ matrix.shardIndex }}
          path: playwright-report/
          retention-days: 30

  # Merge shard reports
  merge-reports:
    if: ${{ !cancelled() }}
    needs: [e2e]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci

      - name: Download all shard reports
        uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: playwright-report-*
          merge-multiple: true

      - name: Merge reports
        run: npx playwright merge-reports --reporter html ./all-blob-reports

      - name: Upload merged report
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

### Test Sharding

```bash
# Run 1 of 4 shards (CI parallelism)
npx playwright test --shard=1/4
npx playwright test --shard=2/4
npx playwright test --shard=3/4
npx playwright test --shard=4/4

# Each shard runs ~25% of tests
# 4 parallel CI runners = ~4× faster
```

### Staging vs Production Strategy

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    // Smoke tests: run on every PR against staging
    {
      name: "smoke",
      testMatch: "**/*.smoke.spec.ts",
      use: { baseURL: process.env.STAGING_URL },
    },
    // Full suite: run on merge to main
    {
      name: "full",
      testIgnore: "**/*.smoke.spec.ts",
      use: { baseURL: process.env.STAGING_URL },
    },
    // Production smoke: run after production deploy
    {
      name: "prod-smoke",
      testMatch: "**/*.smoke.spec.ts",
      use: {
        baseURL: "https://production.example.com",
        // No mocking, real requests, read-only operations only
      },
    },
  ],
});
```

---

## 14. Test Data Management

### Strategies for Test Data

```typescript
// Strategy 1: API setup in beforeEach (recommended for isolated tests)
test.beforeEach(async ({ request }) => {
  // Create test data via API before each test
  await request.post("/api/test/reset-db");
  await request.post("/api/test/seed", {
    data: { users: 1, products: 10, orders: 5 },
  });
});

// Strategy 2: Database seeding via CLI
// package.json: "test:seed": "node scripts/seed-test-db.js"
// Called in CI before tests run

// Strategy 3: Use fixed test accounts (simpler but shared state)
// TEST_USER_EMAIL=testuser@example.com
// This user always exists in staging, tests clean up after themselves
```

### Avoiding Test Data Coupling

```typescript
// ❌ Tests depend on shared mutable data
test("shows product count", async ({ page }) => {
  await page.goto("/products");
  await expect(page.getByTestId("count")).toHaveText("10 products"); // assumes DB has 10
});

// ✅ Tests create their own data or mock the API
test("shows product count", async ({ page }) => {
  await page.route("**/api/products", (route) =>
    route.fulfill({
      body: JSON.stringify(
        Array.from({ length: 10 }, (_, i) => ({ id: i, name: `P${i}` })),
      ),
    }),
  );

  await page.goto("/products");
  await expect(page.getByTestId("count")).toHaveText("10 products");
});
```

### Cleanup After Tests

```typescript
test.afterEach(async ({ request }, testInfo) => {
  if (testInfo.status !== "passed") {
    // Don't clean up on failure — investigate what went wrong
    return;
  }
  // Clean up created resources
  if (createdOrderId) {
    await request.delete(`/api/test/orders/${createdOrderId}`);
  }
});
```

---

## 15. Debugging Failing Tests

### Playwright Inspector (Interactive Debugging)

```bash
# Open Playwright Inspector — step through test interactively
npx playwright test --debug

# Debug a specific test
npx playwright test auth.spec.ts --debug

# Open headed browser (visible UI)
npx playwright test --headed
```

### Trace Viewer

```bash
# Run with tracing
npx playwright test --trace on

# View trace for a specific test result
npx playwright show-trace test-results/auth-login-failed/trace.zip
```

Trace viewer shows:

- Timeline of all actions
- Screenshots at each step
- Network requests and responses
- Console logs
- DOM snapshots you can interact with

### Debugging in Code

```typescript
test("debug example", async ({ page }) => {
  await page.goto("/checkout");

  // Pause test and open inspector
  await page.pause();

  // Take a screenshot at a specific point
  await page.screenshot({ path: "debug-screenshot.png" });

  // Log the current URL
  console.log("Current URL:", page.url());

  // Dump all text on the page
  console.log("Page content:", await page.textContent("body"));

  // Log network requests
  page.on("request", (req) => console.log("Request:", req.method(), req.url()));
  page.on("response", (res) =>
    console.log("Response:", res.status(), res.url()),
  );
});
```

### Retry and Soft Assertions for Flaky Debugging

```typescript
// test.retry: re-runs a failing test to check for flakiness
test.describe.configure({ retries: 3 }); // retry up to 3 times

// Per-test retry
test(
  "potentially flaky test",
  async ({ page }) => {
    // ...
  },
  { retries: 2 },
);
```

---

## 16. Good Practices

### ✅ Test user journeys, not implementation details

```typescript
// ✅ Tests what users care about: can I complete checkout?
test("user can complete a purchase", async ({ page }) => {
  await page.goto("/products");
  await page.getByRole("button", { name: "Add to Cart" }).first().click();
  await page.goto("/checkout");
  // ... complete checkout flow
  await expect(page.getByText("Order Confirmed")).toBeVisible();
});

// ❌ Tests implementation: specific CSS class, DOM structure
test("checkout button has correct class", async ({ page }) => {
  await expect(page.locator(".checkout-btn.primary.large")).toBeVisible();
});
```

### ✅ Use role-based selectors — they test accessibility too

```typescript
// ✅ Role selectors fail if element is inaccessible — catches a11y bugs
await page.getByRole("button", { name: "Submit Order" }).click();
// This fails if there's no accessible "Submit Order" button
// Which means it also tests your ARIA implementation
```

### ✅ Keep tests independent (no shared state)

```typescript
// ✅ Each test creates its own context and data
test.beforeEach(async ({ page }) => {
  // Fresh state for every test
  await page.goto("/");
});
```

### ✅ Name tests as user stories

```typescript
// ✅ Test names describe user behavior
test("user can reset password via email link");
test("admin can deactivate a user account");
test("guest can browse products without logging in");
```

---

## 17. Bad Practices

### ❌ Using CSS classes or IDs as selectors

```typescript
// ❌ Brittle: class names change with CSS refactoring
await page.locator(".btn-primary.checkout-flow__submit").click();

// ✅ Stable: role reflects intent
await page.getByRole("button", { name: "Place Order" }).click();
```

### ❌ Fixed `waitForTimeout` calls

```typescript
// ❌ Never do this — it's always either too short or too long
await page.waitForTimeout(3000);

// ✅ Wait for the condition you actually care about
await expect(page.getByText("Order confirmed")).toBeVisible();
```

### ❌ Testing the same thing at multiple layers

```typescript
// ❌ Form validation tested exhaustively in E2E
test("shows error for empty email"); // unit test this
test("shows error for invalid email"); // unit test this
test("shows error for existing email"); // unit test this
test("shows error for weak password"); // unit test this
// ...20 more validation tests

// ✅ E2E tests the happy path + one representative error
test("form validates on submit");
test("user can sign up successfully");
```

### ❌ Not cleaning up test data

```typescript
// ❌ Tests leave data in the database, affecting other tests
test("creates an order", async ({ page }) => {
  await createOrder(page);
  // Order persists in DB — future tests may see unexpected data
});
```

---

## 18. Common Mistakes

### Mistake 1 — Running E2E tests against production

```
❌ Never run E2E tests that write data against production
   Tests create orders, users, send emails — real consequences

✅ Always run against:
   - Staging environment (mirror of production)
   - Local dev environment
   - Ephemeral environments per PR

Production smoke tests: read-only operations only
  → page.goto('/') → check it loads
  → page.goto('/login') → check form is present
  → No actual login, no data creation
```

### Mistake 2 — Not handling test isolation

```typescript
// ❌ Tests share browser state (cookies, localStorage)
test("logged-in user sees dashboard"); // logs in
test("guest user sees login page"); // expects no auth — but previous test left cookies!

// ✅ Use separate browser contexts or clear state between tests
test.beforeEach(async ({ context }) => {
  await context.clearCookies();
  await context.clearPermissions();
});

// Or use storageState: {} to start clean
```

### Mistake 3 — Asserting too broadly

```typescript
// ❌ Too broad: passes even if page is broken
await expect(page).not.toHaveURL("/error");

// ✅ Assert specific positive outcomes
await expect(page).toHaveURL("/dashboard");
await expect(page.getByRole("heading", { level: 1 })).toHaveText(
  "Your Dashboard",
);
```

### Mistake 4 — Overusing E2E for edge case coverage

```typescript
// ❌ 50 E2E tests for form validation edge cases
// Each takes 3-5 seconds → 50 × 4s = 200 seconds for validation alone

// ✅ 1 E2E test for the happy path
// + 1 E2E test for a representative error
// + 48 unit tests for edge cases (run in <1 second total)
```

---

## 19. Interview-Level Explanation

> **"What is E2E testing? How do you write good E2E tests? How do you handle flakiness?"**

**Strong answer:**

> "E2E testing automates a real browser to verify complete user journeys — from opening a page through completing a workflow and asserting the expected outcome. Unlike unit tests (which verify isolated functions) or integration tests (which verify module interactions), E2E tests verify that the entire stack works together from the user's perspective.
>
> For tooling, Playwright is the modern standard. Its key advantage is automatic waiting — when you click a button or fill an input, Playwright waits for the element to be visible, stable, and enabled before acting. This eliminates most manual `waitFor` calls and a large class of flakiness.
>
> For selector strategy, I follow the testing-library philosophy: prefer selectors that mirror how users experience the page. `getByRole('button', { name: 'Submit' })` is better than `.submit-button` because it only passes if the button has the correct ARIA role and accessible name, which also tests accessibility. If roles aren't sufficient, `data-testid` attributes are the next best choice — explicit test markers that survive CSS refactoring.
>
> The Page Object Model organizes test code by encapsulating page interactions in classes. Tests describe the journey; page objects describe how to interact with the page. This prevents duplication when multiple tests use the same page.
>
> For authentication, the best pattern is to log in once via the API (not the UI), save the browser storage state to a file, and load it for all tests that need auth. This turns a 3-second UI login into a zero-second state restore.
>
> Flakiness usually comes from bad waiting patterns — `waitForTimeout` is almost always wrong. Instead, wait for the specific condition you care about: `waitForURL`, `waitForResponse`, or an `expect` assertion with retry behavior. Playwright's built-in retries combined with CI sharding (running tests in parallel across multiple machines) keeps the full suite fast even at scale."

---

## 20. Exercises

### Exercise 1 — Write a complete test suite

Write E2E tests for this user story: "A user can search for a product, add it to their cart, and see the correct cart total."

Requirements:

- Use role-based selectors
- Handle the async nature of the search (API call)
- Verify cart total updates correctly
- Test adding multiple items

<details>
<summary>Solution skeleton</summary>

```typescript
import { test, expect } from "@playwright/test";

const mockProducts = [
  { id: 1, name: "Wireless Headphones", price: 79.99 },
  { id: 2, name: "USB-C Cable", price: 12.99 },
];

test.describe("Shopping Cart", () => {
  test.beforeEach(async ({ page }) => {
    // Mock the search API
    await page.route("**/api/products/search*", (route) =>
      route.fulfill({
        status: 200,
        contentType: "application/json",
        body: JSON.stringify(mockProducts),
      }),
    );
    // Mock cart API
    await page.route("**/api/cart**", (route) =>
      route.fulfill({
        status: 200,
        body: JSON.stringify({ items: [], total: 0 }),
      }),
    );
  });

  test("user can search for and add a product", async ({ page }) => {
    await page.goto("/");

    const searchInput = page.getByRole("searchbox", {
      name: "Search products",
    });
    await searchInput.fill("headphones");
    await page.keyboard.press("Enter");

    // Wait for results to load
    await expect(
      page.getByRole("list", { name: "Search results" }),
    ).toBeVisible();
    await expect(page.getByRole("listitem")).toHaveCount(2);

    // Add first product
    await page
      .getByRole("listitem")
      .first()
      .getByRole("button", { name: "Add to Cart" })
      .click();

    // Assert cart updated
    await expect(page.getByRole("status", { name: "Cart" })).toContainText("1");
    await expect(page.getByTestId("cart-total")).toContainText("$79.99");
  });

  test("cart total updates correctly for multiple items", async ({ page }) => {
    await page.goto("/");

    // Search and add two products
    await page.getByRole("searchbox").fill("cable");
    await page.keyboard.press("Enter");

    const items = page.getByRole("listitem");
    await items.nth(0).getByRole("button", { name: "Add to Cart" }).click();
    await items.nth(1).getByRole("button", { name: "Add to Cart" }).click();

    // Total should be sum of both
    const expectedTotal = (79.99 + 12.99).toFixed(2);
    await expect(page.getByTestId("cart-total")).toContainText(
      `$${expectedTotal}`,
    );
    await expect(page.getByRole("status", { name: "Cart" })).toContainText("2");
  });
});
```

</details>

---

### Exercise 2 — Identify the flakiness

```typescript
// This test is flaky — fails about 20% of the time in CI. Why, and how do you fix it?
test("notification appears after saving", async ({ page }) => {
  await page.goto("/settings");
  await page.getByLabel("Display Name").fill("Bob");
  await page.getByRole("button", { name: "Save" }).click();
  await page.waitForTimeout(500); // "wait for save"
  await expect(page.getByText("Saved successfully")).toBeVisible();
});
```

<details>
<summary>Answer</summary>

```
Flakiness cause: waitForTimeout(500) is unreliable
  - Network latency in CI varies (sometimes > 500ms)
  - Server under load: response takes 800ms
  - Result: notification not visible yet when assertion runs

Fix: wait for the actual condition instead of a fixed time

  Option 1: waitForResponse (most explicit)
    const [response] = await Promise.all([
      page.waitForResponse('**/api/settings'),
      page.getByRole('button', { name: 'Save' }).click(),
    ]);
    expect(response.status()).toBe(200);
    await expect(page.getByText('Saved successfully')).toBeVisible();

  Option 2: rely on expect retry (simplest fix)
    await page.getByRole('button', { name: 'Save' }).click();
    // Remove waitForTimeout completely
    // expect retries until visible or timeout (default 5000ms)
    await expect(page.getByText('Saved successfully')).toBeVisible();

  Option 3: wait for loading state to clear
    await page.getByRole('button', { name: 'Save' }).click();
    await expect(page.getByRole('button', { name: 'Save' })).toBeEnabled();
    // Button re-enables when save completes
    await expect(page.getByText('Saved successfully')).toBeVisible();
```

</details>

---

## 🔗 Related Topics

- [`testing/01-unit-testing.md`](./01-unit-testing.md) — Unit testing fundamentals
- [`testing/02-integration-testing.md`](./02-integration-testing.md) — Integration testing with Testing Library
- [`debugging/01-browser-devtools.md`](../debugging/01-browser-devtools.md) — DevTools for debugging E2E failures
- [`browser-internals/02-dom-tree-creation.md`](../browser-internals/02-dom-tree-creation.md) — Understanding what Playwright interacts with

---

<div align="center">

**`testing/` section:** [`01-unit-testing.md`](./01-unit-testing.md) · [`02-integration-testing.md`](./02-integration-testing.md) · **`03-e2e-testing.md`** ✓

</div>
