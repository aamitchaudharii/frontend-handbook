# 01 — Prop Drilling

> **"Prop drilling isn't wrong because it breaks anything. It's wrong because it makes the code fragile in a specific way: every intermediate component becomes a silent participant in a data contract it didn't ask for and doesn't benefit from. Change the shape of that data and you touch every layer, including the layers that were just passing it along."**

Prop drilling is the practice of passing props through multiple layers of components solely to deliver data to a deeply nested consumer, with every intermediate component acting as a conduit rather than a genuine user of the data. It starts benign — one or two levels is completely normal — and becomes a maintenance problem when refactoring forces you to touch five files just to rename a prop that only one component actually uses. This document covers how to recognize problematic prop drilling, the four solutions available (composition, Context, state managers, colocation), and how to choose the right one.

---

## 📚 Table of Contents

1. [What Prop Drilling Is and Isn't](#1-what-prop-drilling-is-and-isnt)
2. [Recognizing the Problem](#2-recognizing-the-problem)
3. [Solution 1 — Composition (Inversion of Control)](#3-solution-1--composition-inversion-of-control)
4. [Solution 2 — React Context](#4-solution-2--react-context)
5. [Solution 3 — External State Manager](#5-solution-3--external-state-manager)
6. [Solution 4 — Component Colocation](#6-solution-4--component-colocation)
7. [Choosing the Right Solution](#7-choosing-the-right-solution)
8. [Context Pitfalls to Avoid](#8-context-pitfalls-to-avoid)
9. [Good Practices](#9-good-practices)
10. [Bad Practices](#10-bad-practices)
11. [Common Mistakes](#11-common-mistakes)
12. [Interview-Level Explanation](#12-interview-level-explanation)
13. [Exercises](#13-exercises)

---

## 1. What Prop Drilling Is and Isn't

```
NOT PROP DRILLING (normal, healthy):
  <Parent data={data}>
    <Child data={data} />   ← 1 level, Child genuinely uses data
  </Parent>

MILD PROP DRILLING (usually acceptable):
  <GrandParent data={data}>
    <Parent data={data}>    ← passes data through
      <Child data={data} /> ← 2 levels, Child genuinely uses data
    </Parent>
  </GrandParent>

PROBLEMATIC PROP DRILLING (3+ levels, intermediaries don't use the data):
  <App user={user}>
    <Layout user={user}>        ← doesn't use user, just passes it
      <Sidebar user={user}>     ← doesn't use user, just passes it
        <Navigation user={user}> ← doesn't use user, just passes it
          <UserMenu user={user} /> ← FINALLY uses user
        </Navigation>
      </Sidebar>
    </Layout>
  </App>

  Changing User type → must update App, Layout, Sidebar, Navigation, UserMenu
  Adding a field to user → must thread through 4 files
  Renaming `user` prop → 4 files to change
```

---

## 2. Recognizing the Problem

```javascript
// Smell 1: Props present in component signature but NOT used in render
function Layout({ children, user, theme, locale, permissions }) {
  //                         ^^^^  ^^^^^  ^^^^^^  ^^^^^^^^^^^  none of these
  //                         are used in Layout's own JSX — all just forwarded
  return (
    <div className="layout">
      <Sidebar
        user={user}
        theme={theme}
        locale={locale}
        permissions={permissions}
      />
      <main>{children}</main>
    </div>
  );
}

// Smell 2: Same prop appearing in 4+ consecutive component files
// Smell 3: Updating a data shape requires touching intermediate components
// Smell 4: Adding a new consumer requires threading a prop through layers

// MEASURING PROP DRILLING:
// Quick heuristic: if you grep for a prop name and it appears in 3+ files
// but is only *used* (not just forwarded) in 1 file — you have prop drilling.
```

---

## 3. Solution 1 — Composition (Inversion of Control)

Composition is the most underused solution. Instead of passing data down through intermediaries, pass the already-rendered element:

```jsx
// ❌ BEFORE: drilling `user` through Layout and Sidebar
function App() {
  const user = useCurrentUser();
  return <Layout user={user} />;
}

function Layout({ user }) {
  // doesn't use `user` directly
  return (
    <div>
      <Sidebar user={user} />
      <main>...</main>
    </div>
  );
}

function Sidebar({ user }) {
  // doesn't use `user` directly
  return (
    <nav>
      <UserMenu user={user} />
    </nav>
  );
}

function UserMenu({ user }) {
  return <div>{user.name}</div>; // ← only component that uses user
}
```

```jsx
// ✅ AFTER: compose UserMenu where user is in scope, pass the element down
function App() {
  const user = useCurrentUser();
  return (
    <Layout sidebar={<Sidebar nav={<UserMenu user={user} />} />}>
      main content
    </Layout>
  );
}

function Layout({ children, sidebar }) {
  // No user prop — it's in the rendered element, not the props
  return (
    <div>
      {sidebar}
      <main>{children}</main>
    </div>
  );
}

function Sidebar({ nav }) {
  // No user prop
  return <nav>{nav}</nav>;
}

function UserMenu({ user }) {
  return <div>{user.name}</div>;
}

// Result:
// - Layout and Sidebar have NO knowledge of user
// - Adding fields to user: change App and UserMenu only
// - Renaming user prop: change App and UserMenu only
// - Layout and Sidebar are now independently reusable
```

---

## 4. Solution 2 — React Context

Context provides implicit data access for genuinely cross-cutting data needed by many unrelated parts of the tree:

```jsx
// Create a Context with sensible default handling
const UserContext = createContext(null);

function useUser() {
  const user = useContext(UserContext);
  if (!user) throw new Error("useUser must be used within UserProvider");
  return user;
}

// Provider: wraps the part of the tree that needs user data
function UserProvider({ children }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ["currentUser"],
    queryFn: getCurrentUser,
  });
  if (isLoading) return <FullPageSpinner />;
  return <UserContext.Provider value={user}>{children}</UserContext.Provider>;
}

// Any component at any depth can now access user without prop drilling
function UserMenu() {
  const user = useUser(); // no props needed
  return <div>{user.name}</div>;
}

function UserAvatar() {
  const user = useUser(); // works in any nested component
  return <img src={user.avatarUrl} alt={user.name} />;
}

// Parent components remain completely ignorant of user
function Layout({ children, sidebar }) {
  return (
    <div>
      {sidebar}
      <main>{children}</main>
    </div>
  ); // no user prop
}
```

### Performance: Splitting Context by Update Frequency

```jsx
// ❌ One large context causes ALL consumers to re-render on ANY change
const AppContext = createContext(null);
// If notifications count changes → user profile consumers re-render too

// ✅ Split by how often data changes
const UserContext = createContext(null); // changes infrequently (on login/logout)
const NotificationsContext = createContext(null); // changes frequently (new notifications)
const ThemeContext = createContext(null); // changes on user preference toggle

// Each context change only re-renders components subscribed to THAT context
```

---

## 5. Solution 3 — External State Manager

For state that needs to be accessed by many components across the entire app, external stores (Zustand, Redux) provide direct access without a Context tree:

```typescript
// Zustand: components access exactly what they need, no prop drilling
import { create } from 'zustand';

const useUserStore = create((set) => ({
  user: null,
  setUser: (user) => set({ user }),
}));

// Any component, at any depth, accesses user directly:
function UserMenu() {
  const user = useUserStore(s => s.user); // subscribes only to user slice
  return <div>{user?.name}</div>;
}

function Avatar() {
  const avatarUrl = useUserStore(s => s.user?.avatarUrl); // subscribes to avatarUrl only
  return <img src={avatarUrl} />;
}

// No prop drilling, no Provider needed for simple stores
// Re-renders: only when the selected slice changes
```

---

## 6. Solution 4 — Component Colocation

Sometimes the real fix is moving state CLOSER to where it's used, so it no longer needs to travel:

```jsx
// ❌ BEFORE: user settings state lives in App, gets drilled down
function App() {
  const [theme, setTheme] = useState("light");
  const [fontSize, setFontSize] = useState(16);
  // theme and fontSize drilled through Layout → Sidebar → SettingsPanel
  return (
    <Layout
      theme={theme}
      setTheme={setTheme}
      fontSize={fontSize}
      setFontSize={setFontSize}
    />
  );
}

// ✅ AFTER: settings state moves to the component that owns it
// App doesn't need to know about theme or fontSize at all
function SettingsPanel() {
  // State lives HERE — no drilling needed because it never leaves this component
  const [theme, setTheme] = useState("light");
  const [fontSize, setFontSize] = useState(16);

  return (
    <div>
      <ThemeSelector value={theme} onChange={setTheme} />
      <FontSizeControl value={fontSize} onChange={setFontSize} />
    </div>
  );
}
```

---

## 7. Choosing the Right Solution

```
DECISION FRAMEWORK:

1. Does the intermediate component have any RIGHT to know about this data?
   NO → composition (pass the rendered element, skip the intermediary)
   YES → continue below

2. Is the data needed by one component or a small, tightly related group?
   YES → just pass props (1-2 levels is fine)

3. Is the data needed by many UNRELATED components across the tree?
   YES → React Context or external store

4. Does the state really belong in the parent? Or has it just drifted up?
   State drifted up → colocate state closer to where it's used

PATTERNS BY USE CASE:
  Structural data (layout slots): composition
  Theme, locale, auth user: Context (changes infrequently, broad access)
  Server state (products, users): TanStack Query (co-locates with consumers)
  UI state used by 1 component: colocate in that component
  Complex client state across features: Zustand/Redux
```

---

## 8. Context Pitfalls to Avoid

```jsx
// ❌ Pitfall 1: Putting everything in one Context
const AppContext = createContext({
  user,
  theme,
  locale,
  cart,
  notifications,
  settings,
  featureFlags,
});
// Any change → EVERYTHING re-renders

// ❌ Pitfall 2: New object every render
function Provider({ children }) {
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {" "}
      // new obj every render!
      {children}
    </ThemeContext.Provider>
  );
}
// ✅ Fix: memoize context value
const value = useMemo(() => ({ theme, setTheme }), [theme, setTheme]);

// ❌ Pitfall 3: Context for state that could be colocated
// Using Context for a Dialog's open state when only the parent + Dialog need it
const DialogContext = createContext(); // overkill — just pass onClose as a prop

// ❌ Pitfall 4: Context for frequently-changing values (e.g., scroll position)
const ScrollContext = createContext();
// Context change → all consumers re-render → triggers 60fps re-renders on scroll
// ✅ Fix: use a ref + direct DOM manipulation for high-frequency values
```

---

## 9. Good Practices

### ✅ Identify the real consumer before deciding a solution

```
Before reaching for Context: ask who actually USES this data.
If one component uses it: colocate state there.
If a few related components use it: pass props (normal).
If many unrelated components use it: Context or external store.
```

### ✅ Try composition before Context for structural data

```jsx
// Composition often eliminates prop drilling without adding Context complexity
<Layout topbar={<Topbar user={user} />}>
  {" "}
  // pass the element
  <Page />
</Layout>
// Layout never needs to know about user
```

### ✅ Colocate state as close to its consumers as possible

```jsx
// ✅ State that only a subtree needs should NOT live at App root
function ExpandableSection({ title, children }) {
  const [isOpen, setIsOpen] = useState(false); // colocated
  return (/* ... */);
}
```

---

## 10. Bad Practices

### ❌ Spreading all parent props to children to avoid listing them

```jsx
// ❌ Props spreading "solves" drilling but hides the data flow entirely
function Parent(props) {
  return <Child {...props} />; // what does Child actually receive?
}
function Child(props) {
  return <GrandChild {...props} />; // complete loss of visibility
}
// Worse than explicit prop drilling — at least drilling is traceable
```

### ❌ Creating a massive App-level Context for all state

```jsx
// ❌ Kitchen-sink Context: everything re-renders on any change
<AppContext.Provider value={{ user, theme, cart, settings, notifications }}>
  <App />
</AppContext.Provider>
```

---

## 11. Common Mistakes

### Mistake 1 — Using Context when composition would be simpler

```jsx
// ❌ Context for data that only needs to travel 1-2 levels
const ButtonGroupContext = createContext();
function ButtonGroup({ size }) {
  return (
    <ButtonGroupContext.Provider value={size}>
      <div>...</div>
    </ButtonGroupContext.Provider>
  );
}
// This is context for passing `size` to direct children — just pass it as a prop!

// ✅ Simply pass size as a prop to direct children
function ButtonGroup({ size, children }) {
  return (
    <div>
      {React.Children.map(children, (child) =>
        React.cloneElement(child, { size }),
      )}
    </div>
  );
}
// Or: pass the rendered buttons as children with size already applied
```

### Mistake 2 — Not noticing state that should be colocated

```jsx
// ❌ UI state drifted to root for no real reason
function App() {
  const [isMenuOpen, setIsMenuOpen] = useState(false); // used only in NavMenu
  return <Layout isMenuOpen={isMenuOpen} setIsMenuOpen={setIsMenuOpen} />;
}

// ✅ Colocate: NavMenu owns its own open state
function NavMenu() {
  const [isOpen, setIsOpen] = useState(false); // stays here
  return (/* ... */);
}
```

---

## 12. Interview-Level Explanation

> **"What is prop drilling? How do you solve it?"**

**Strong answer:**

> "Prop drilling is when data is passed through multiple layers of components as props, with intermediate components acting only as conduits — they receive the prop but don't use it themselves, just forwarding it to their children. It's a maintenance problem: when the data shape changes, you have to update every intermediate component, even the ones that don't actually care about the data.
>
> The first solution people reach for is Context, but it's not always the right first choice. Before adding Context, I ask: does the intermediate component have any legitimate reason to know about this data? If not, composition often solves it more elegantly. Instead of drilling `user` through Layout → Sidebar → Navigation → UserMenu, you can construct the UserMenu element where `user` is in scope — at the App level — and pass the rendered element down through the tree. Layout, Sidebar, and Navigation never see `user`; they just slot the element in. This is the inversion of control pattern — instead of drilling data down to a leaf, you lift the leaf's rendering up to where the data is.
>
> When you genuinely need many unrelated components across the tree to access the same data — authentication state, theme, locale — Context is the right tool. The key is splitting Context by update frequency: auth data (changes rarely) in one Context, notification count (changes frequently) in another. A single large Context causes every subscriber to re-render whenever any part of it changes.
>
> For complex client state or server state accessed across many features, external stores like Zustand or TanStack Query let components subscribe directly to exactly the slice they need, no Provider required and with very precise re-render control.
>
> The fourth and most often overlooked solution is colocation: state that only one component needs often ends up at the App root through gradual drift — someone lifted it 'just in case' and it stayed there. Moving it back down eliminates the drilling entirely and makes the component self-contained."

---

## 13. Exercises

### Exercise 1 — Refactor prop drilling

```jsx
// Refactor this prop-drilled tree using composition and/or Context.
// Identify which solution is appropriate for each type of data.

function App() {
  const user = useCurrentUser(); // needed by: Header/Avatar, UserMenu, Profile
  const theme = useTheme(); // needed by: ThemeToggle only
  const cartCount = useCartCount(); // needed by: CartIcon only

  return (
    <Layout user={user} theme={theme} cartCount={cartCount}>
      <MainContent user={user} />
    </Layout>
  );
}

function Layout({ children, user, theme, cartCount }) {
  return (
    <div>
      <Header user={user} theme={theme} cartCount={cartCount} />
      <main>{children}</main>
    </div>
  );
}

function Header({ user, theme, cartCount }) {
  return (
    <header>
      <Logo />
      <UserAvatar user={user} />
      <ThemeToggle theme={theme} />
      <CartIcon count={cartCount} />
    </header>
  );
}

function MainContent({ user }) {
  return <Profile user={user} />;
}
```

<details>
<summary>Solution</summary>

```jsx
// ANALYSIS:
// user: needed by UserAvatar, Profile, UserMenu → Context (broad, cross-tree access)
// theme: ThemeToggle owns it → colocate state in ThemeToggle
// cartCount: CartIcon owns it → colocate fetching in CartIcon

// 1. User via Context (needed broadly, infrequently changes)
const UserContext = createContext(null);
function UserProvider({ children }) {
  const user = useCurrentUser();
  return <UserContext.Provider value={user}>{children}</UserContext.Provider>;
}
function useUser() {
  const user = useContext(UserContext);
  if (!user) throw new Error("useUser must be used within UserProvider");
  return user;
}

// 2. Theme colocated in ThemeToggle (only this component needs it)
function ThemeToggle() {
  const [theme, setTheme] = useTheme(); // colocated hook
  return <button onClick={setTheme}>Toggle</button>;
}

// 3. CartIcon owns its own data fetching
function CartIcon() {
  const cartCount = useCartCount(); // colocated
  return <div>🛒 {cartCount}</div>;
}

// 4. UserAvatar and Profile read from Context directly
function UserAvatar() {
  const user = useUser();
  return <img src={user.avatarUrl} />;
}
function Profile() {
  const user = useUser();
  return <div>{user.name}</div>;
}

// RESULT: App, Layout, and Header have no knowledge of user, theme, or cartCount
function App() {
  return (
    <UserProvider>
      <Layout>
        <Profile /> {/* reads user from Context */}
      </Layout>
    </UserProvider>
  );
}
function Layout({ children }) {
  return (
    <div>
      <Header /> {/* no props needed */}
      <main>{children}</main>
    </div>
  );
}
function Header() {
  return (
    <header>
      <Logo />
      <UserAvatar /> {/* reads user from Context */}
      <ThemeToggle /> {/* owns its own state */}
      <CartIcon /> {/* owns its own data */}
    </header>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`patterns/01-component-composition.md`](../patterns/01-component-composition.md) — Composition as the first fix
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md) — Hooks for shared logic without drilling
- [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md) — Where state should live

---

<div align="center">

**Next:** [`anti-patterns/02-god-components.md`](./02-god-components.md) →

</div>
