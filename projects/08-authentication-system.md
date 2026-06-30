# 08 — Project: Authentication System (Frontend)

> **"Authentication is the one feature where 'good enough' is dangerous — a subtle flaw here doesn't degrade the user experience, it compromises every user's account. Building it once, correctly, with a clear mental model of token lifecycle and storage, is the difference between a system that's secure by construction and one that's secure by luck."**

This project guide builds a complete frontend authentication system: login/signup flows, token storage and refresh, protected routes, persistent sessions, multi-factor authentication, and social login (OAuth) integration — applying the security patterns from [`security/04-auth-patterns.md`](../security/04-auth-patterns.md) in a complete, working system.

---

## 📚 What You'll Build

An authentication system with: email/password login and signup with validation, secure token storage (access token in memory, refresh token in HttpOnly cookie), automatic silent token refresh, protected route guarding, "remember me" persistent sessions, MFA challenge flow, and social login (Google/GitHub OAuth) — all integrated into a cohesive auth context.

---

## Requirements

```
FUNCTIONAL:
  - Email/password registration with validation (email format, password
    strength) and clear error messaging
  - Login with "remember me" option
  - Password reset flow (request → email link → set new password)
  - Multi-factor authentication challenge after password verification
  - Social login via OAuth (Google, GitHub)
  - Protected routes that redirect unauthenticated users to login,
    then redirect back to their original destination after login
  - Logout that fully clears client and server session state

NON-FUNCTIONAL:
  - Access tokens never touch localStorage/sessionStorage (XSS mitigation)
  - Silent token refresh happens transparently — users are never logged
    out mid-session due to a routine token expiry
  - No flash of incorrect UI (logged-out content briefly shown to a
    logged-in user, or vice versa) during the initial auth check
```

---

## Architecture Overview

```
COMPONENT TREE:
  <AuthProvider>                 (wraps the entire app, owns auth state)
    <App>
      <Routes>
        <Route path="/login" element={<LoginPage />} />
        <Route path="/signup" element={<SignupPage />} />
        <Route path="/forgot-password" element={<ForgotPasswordPage />} />
        <Route element={<RequireAuth />}>     (layout route guarding children)
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
        </Route>

STATE LAYERS (see security/04-auth-patterns.md for the full rationale):
  1. Access token: in-memory only (module-level variable / class instance)
  2. Refresh token: HttpOnly, Secure, SameSite cookie (never touched by JS)
  3. User profile: React state, populated after token validation
  4. Auth status: 'loading' | 'authenticated' | 'unauthenticated'
```

---

## Step 1 — The Auth Context (Tying Together Storage, Refresh, and User State)

```typescript
interface AuthContextValue {
  user:        User | null;
  status:      'loading' | 'authenticated' | 'unauthenticated';
  login:       (credentials: LoginCredentials) => Promise<LoginResult>;
  signup:      (data: SignupData) => Promise<void>;
  logout:      () => Promise<void>;
  refreshUser: () => Promise<void>;
}

const AuthContext = createContext<AuthContextValue | null>(null);

function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser]     = useState<User | null>(null);
  const [status, setStatus] = useState<'loading' | 'authenticated' | 'unauthenticated'>('loading');

  // On mount: attempt silent refresh using the HttpOnly refresh token cookie
  // This determines whether the user has an existing valid session
  useEffect(() => {
    async function checkExistingSession() {
      try {
        const { accessToken, expiresIn, user } = await authService.silentRefresh();
        tokenStore.setToken(accessToken, expiresIn);
        refreshScheduler.schedule(expiresIn);
        setUser(user);
        setStatus('authenticated');
      } catch {
        setStatus('unauthenticated'); // no valid session — that's fine, not an error state
      }
    }
    checkExistingSession();
  }, []);

  const login = useCallback(async (credentials: LoginCredentials): Promise<LoginResult> => {
    const response = await authApi.login(credentials); // credentials: 'include' for cookie receipt

    if (response.requiresMfa) {
      return { status: 'mfa_required', mfaSessionId: response.mfaSessionId };
    }

    tokenStore.setToken(response.accessToken, response.expiresIn);
    refreshScheduler.schedule(response.expiresIn);
    setUser(response.user);
    setStatus('authenticated');
    return { status: 'success' };
  }, []);

  const logout = useCallback(async () => {
    await authApi.logout(); // server invalidates the refresh token
    tokenStore.clear();
    refreshScheduler.cancel();
    setUser(null);
    setStatus('unauthenticated');
  }, []);

  const signup = useCallback(async (data: SignupData) => {
    await authApi.signup(data);
    // Most systems require email verification before login — don't
    // auto-login here; redirect to a "check your email" screen instead
  }, []);

  const refreshUser = useCallback(async () => {
    const updated = await authApi.getCurrentUser();
    setUser(updated);
  }, []);

  return (
    <AuthContext.Provider value={{ user, status, login, signup, logout, refreshUser }}>
      {children}
    </AuthContext.Provider>
  );
}

function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

**Key decision:** the initial auth check on mount attempts a SILENT refresh (using the HttpOnly cookie, no UI shown) rather than immediately redirecting to the login page or assuming the user is logged out. This is what makes persistent sessions (staying logged in across browser restarts) work correctly — the `status: 'loading'` state covers this brief window, and components must wait for it to resolve before deciding what to render (Step 4 covers this).

---

## Step 2 — Login Form with MFA Branching

```jsx
function LoginPage() {
  const { login } = useAuth();
  const navigate   = useNavigate();
  const location   = useLocation();
  const from = (location.state as { from?: Location })?.from?.pathname ?? '/dashboard';

  const [mfaSession, setMfaSession] = useState<{ sessionId: string } | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isSubmitting, setSubmitting] = useState(false);

  async function handleLogin(credentials: LoginCredentials) {
    setSubmitting(true);
    setError(null);
    try {
      const result = await login(credentials);

      if (result.status === 'mfa_required') {
        setMfaSession({ sessionId: result.mfaSessionId });
      } else {
        navigate(from, { replace: true }); // back to where they were headed
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed');
    } finally {
      setSubmitting(false);
    }
  }

  function handleMfaSuccess() {
    navigate(from, { replace: true });
  }

  if (mfaSession) {
    return <MfaChallenge sessionId={mfaSession.sessionId} onSuccess={handleMfaSuccess} />;
  }

  return (
    <LoginForm onSubmit={handleLogin} error={error} isSubmitting={isSubmitting} />
  );
}
```

**Key decision:** the `from` location is read from `location.state`, which is how [`RequireAuth`](#step-4--protected-route-guard) communicates "this is where the user was trying to go before being redirected to login" — preserving the original destination across the login (and possible MFA) flow is what makes "log in and land back where you were" work, rather than always dumping the user on a generic dashboard after login regardless of what link they originally clicked.

---

## Step 3 — MFA Challenge Component

```jsx
function MfaChallenge({ sessionId, onSuccess }) {
  const [code, setCode]     = useState('');
  const [error, setError]   = useState<string | null>(null);
  const [isVerifying, setVerifying] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setVerifying(true);
    setError(null);

    try {
      const response = await authApi.verifyMfa({ sessionId, code });
      tokenStore.setToken(response.accessToken, response.expiresIn);
      refreshScheduler.schedule(response.expiresIn);
      onSuccess();
    } catch (err) {
      setError('Invalid code. Please try again.');
      setCode('');
    } finally {
      setVerifying(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>Two-Factor Authentication</h2>
      <p>Enter the 6-digit code from your authenticator app</p>
      <input
        value={code}
        onChange={e => setCode(e.target.value.replace(/\D/g, '').slice(0, 6))}
        inputMode="numeric"
        autoComplete="one-time-code"
        autoFocus
        maxLength={6}
      />
      {error && <p role="alert" className="error">{error}</p>}
      <button type="submit" disabled={code.length !== 6 || isVerifying}>
        {isVerifying ? 'Verifying...' : 'Verify'}
      </button>
    </form>
  );
}
```

**Key decision:** `autoComplete="one-time-code"` is a small but meaningful detail — it allows mobile browsers and password managers to automatically suggest/fill the code from an SMS message on supported platforms, reducing friction in what is otherwise an extra manual step in the login flow.

---

## Step 4 — Protected Route Guard

```jsx
function RequireAuth() {
  const { status } = useAuth();
  const location = useLocation();

  if (status === "loading") {
    // CRITICAL: don't render anything definitive yet — avoids a flash
    // of either the protected content OR a premature redirect to login
    // while the silent refresh check (Step 1) is still in flight
    return <FullPageSpinner />;
  }

  if (status === "unauthenticated") {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />; // renders the matched child route
}

// Router configuration
function AppRouter() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/signup" element={<SignupPage />} />

      <Route element={<RequireAuth />}>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Route>
    </Routes>
  );
}
```

**Key decision:** the three-state `status` model (`loading` / `authenticated` / `unauthenticated`) rather than a simple boolean `isLoggedIn` is what prevents the "flash of wrong content" problem — a boolean defaulting to `false` would briefly redirect an actually-logged-in user (with a valid refresh token) to the login page before the silent refresh check completes, only to bounce them back to the dashboard a moment later. The explicit `loading` state lets `RequireAuth` correctly wait before making any redirect decision.

---

## Step 5 — Social Login (OAuth) Integration

```typescript
// Using the PKCE flow from security/04-auth-patterns.md
function useSocialLogin() {
  async function loginWithGoogle() {
    const { verifier, challenge } = await generatePKCE();
    const state = generateRandomString(32);

    sessionStorage.setItem('oauth_verifier', verifier);
    sessionStorage.setItem('oauth_state', state);

    const params = new URLSearchParams({
      client_id:             GOOGLE_CLIENT_ID,
      redirect_uri:          `${window.location.origin}/auth/callback/google`,
      response_type:         'code',
      scope:                 'openid email profile',
      state,
      code_challenge:        challenge,
      code_challenge_method: 'S256',
    });

    window.location.href = `https://accounts.google.com/o/oauth2/v2/auth?${params}`;
  }

  return { loginWithGoogle };
}

// OAuth callback handler page
function OAuthCallbackPage({ provider }: { provider: 'google' | 'github' }) {
  const navigate = useNavigate();
  const { refreshUser } = useAuth();
  const [searchParams] = useSearchParams();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function handleCallback() {
      const code  = searchParams.get('code');
      const state = searchParams.get('state');
      const storedState = sessionStorage.getItem('oauth_state');
      const verifier     = sessionStorage.getItem('oauth_verifier');

      sessionStorage.removeItem('oauth_state');
      sessionStorage.removeItem('oauth_verifier');

      if (!code || state !== storedState) {
        setError('Authentication failed — invalid state. Please try again.');
        return;
      }

      try {
        // Exchange happens server-side (client secret never touches the browser)
        const response = await authApi.exchangeOAuthCode({ provider, code, verifier });
        tokenStore.setToken(response.accessToken, response.expiresIn);
        refreshScheduler.schedule(response.expiresIn);
        await refreshUser();
        navigate('/dashboard', { replace: true });
      } catch {
        setError('Failed to complete sign-in. Please try again.');
      }
    }
    handleCallback();
  }, [searchParams, provider, navigate, refreshUser]);

  if (error) return <AuthErrorScreen message={error} onRetry={() => navigate('/login')} />;
  return <FullPageSpinner label="Completing sign-in..." />;
}
```

**Key decision:** `state` validation happens BEFORE any other processing in the callback — if the returned `state` doesn't match what was stored before redirecting to the provider, the flow is aborted immediately. This is the CSRF protection mechanism for OAuth flows; without it, an attacker could potentially trick a user's browser into completing an OAuth flow initiated by the attacker, associating the attacker's account with the victim's session.

---

## Step 6 — Logout (Full Cleanup)

```typescript
async function logout() {
  // 1. Tell the server to invalidate the refresh token (server-side state)
  await authApi.logout(); // server clears the HttpOnly cookie + revokes the DB session record

  // 2. Clear client-side in-memory state
  tokenStore.clear();
  refreshScheduler.cancel();

  // 3. Clear any cached user/app data that shouldn't persist across users
  //    on a SHARED device (e.g., a public computer)
  queryClient.clear(); // clear ALL TanStack Query cache, not just auth-related

  // 4. Update auth state — triggers re-render, RequireAuth redirects to login
  setUser(null);
  setStatus("unauthenticated");

  // 5. Navigate explicitly (in case the user is on a page that doesn't
  //    automatically redirect via RequireAuth, e.g., a public page)
  window.location.href = "/login"; // full reload, not client-side navigate —
  // ensures ALL in-memory state (including
  // anything outside React's control) is reset
}
```

**Key decision:** logout uses `window.location.href` (a full page reload) rather than a client-side route navigation — this guarantees that absolutely no stale in-memory state survives the logout, which matters significantly on shared/public devices where the NEXT person to use the browser must not see any trace of the previous user's session, cached data, or application state.

---

## Security Checklist

```
☐ Access token stored in memory only — never localStorage/sessionStorage
☐ Refresh token in HttpOnly, Secure, SameSite=Strict cookie
☐ OAuth flows use PKCE + state parameter validation
☐ MFA challenge happens server-side; client never has a way to bypass it
☐ Logout clears server-side session AND all client-side state
☐ No sensitive data (passwords, full tokens) ever logged to console,
  even in development
☐ Password reset tokens are single-use and time-limited (enforced server-side)
☐ Rate limiting on login attempts (enforced server-side, but UI should
  handle and display the resulting error gracefully)
```

## UX Checklist

```
☐ No flash of wrong content during the initial silent-refresh check
☐ "Remember me" / persistent session behavior clearly understood by the user
☐ Login redirects back to the originally requested page after success
☐ Clear, specific error messages (without revealing whether an EMAIL
  exists in the system, which would be an account enumeration vulnerability —
  "Invalid email or password" rather than "No account with that email")
☐ MFA input supports paste and auto-fill from SMS where applicable
```

---

## Extension Ideas

```
- Passkey/WebAuthn support for passwordless login
- Account linking (let a user connect both Google login AND email/password
  to the same account)
- Session management UI (view and revoke active sessions/devices)
- Suspicious login detection with email notifications
- Progressive profiling (collect additional info after initial signup,
  rather than a long upfront form)
```

---

## 🔗 Related Topics

- [`security/04-auth-patterns.md`](../security/04-auth-patterns.md) — Full security rationale for every pattern used here
- [`security/01-xss.md`](../security/01-xss.md) — Why token storage location matters
- [`security/02-csrf.md`](../security/02-csrf.md) — CSRF considerations for cookie-based auth

---

<div align="center">

**Next:** [`projects/09-notification-system.md`](./09-notification-system.md) →

</div>
