# 03 — Render Props and Higher-Order Components

> **"Render props and HOCs are solutions to a problem hooks solved better — but they didn't disappear. HOCs are the canonical way to integrate third-party libraries with class components, and they still appear throughout codebases that can't migrate. Render props remain powerful when you need to thread state into deeply nested JSX structure. Understanding both patterns is understanding the problems they were designed for."**

Render props and Higher-Order Components (HOCs) are the pre-hooks patterns for sharing stateful logic between React components. HOCs wrap a component in another component that injects additional props. Render props delegate rendering control to a function prop. Both patterns remain relevant: HOCs are ubiquitous in library APIs (`connect()`, `withRouter()`, `withStyles()`), and render props appear in complex slot-based APIs where the caller must control the structure around shared state. This document covers both patterns, their tradeoffs, the wrapper hell problem, and when each still makes sense today.

---

## 📚 Table of Contents

1. [The Problem Both Patterns Solve](#1-the-problem-both-patterns-solve)
2. [Higher-Order Components](#2-higher-order-components)
3. [HOC Composition and Naming](#3-hoc-composition-and-naming)
4. [HOC Pitfalls](#4-hoc-pitfalls)
5. [Render Props](#5-render-props)
6. [Render Props vs Custom Hooks](#6-render-props-vs-custom-hooks)
7. [The Wrapper Hell Problem](#7-the-wrapper-hell-problem)
8. [When HOCs Still Make Sense](#8-when-hocs-still-make-sense)
9. [When Render Props Still Make Sense](#9-when-render-props-still-make-sense)
10. [Migrating from HOC/Render Props to Hooks](#10-migrating-from-hoc-render-props-to-hooks)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. The Problem Both Patterns Solve

```jsx
// THE CROSS-CUTTING CONCERN PROBLEM (pre-hooks):
// You want to share STATEFUL LOGIC (state + effects) between components
// without duplicating the code in each.

// Example: 5 components all need "is this element visible on screen?"
// Options before hooks:
//   1. Copy-paste the IntersectionObserver logic into each component (bad)
//   2. Put it in a base class, extend it (inheritance — bad in React)
//   3. Higher-order component: withVisibility(Component) (HOC approach)
//   4. Render prop: <Visibility>{({ isVisible }) => <Component />}</Visibility>

// Both HOC and render prop solve the same problem:
// Extract the "is visible?" logic into one place, inject it into any component
// that needs it — WITHOUT that component knowing the implementation details.

// Modern answer: custom hook `useInView()` — much simpler, both patterns are
// largely superseded. But they remain important to understand for:
//   - Existing codebases (millions of projects still use them)
//   - Third-party library APIs (react-redux's connect, react-router's withRouter)
//   - Specific scenarios where hooks genuinely can't express the pattern cleanly
```

---

## 2. Higher-Order Components

A HOC is a function that takes a component and returns a new, enhanced component:

```javascript
// HOC signature: (Component) => EnhancedComponent
// or with config: (config) => (Component) => EnhancedComponent

// Simple HOC: inject "isOnline" prop into any component
function withOnlineStatus(WrappedComponent) {
  function WithOnlineStatus(props) {
    const [isOnline, setIsOnline] = React.useState(navigator.onLine);

    React.useEffect(() => {
      const goOnline = () => setIsOnline(true);
      const goOffline = () => setIsOnline(false);
      window.addEventListener("online", goOnline);
      window.addEventListener("offline", goOffline);
      return () => {
        window.removeEventListener("online", goOnline);
        window.removeEventListener("offline", goOffline);
      };
    }, []);

    return <WrappedComponent {...props} isOnline={isOnline} />;
  }

  // Convention: name the HOC for DevTools
  WithOnlineStatus.displayName = `WithOnlineStatus(${getDisplayName(WrappedComponent)})`;
  return WithOnlineStatus;
}

function getDisplayName(Component) {
  return Component.displayName || Component.name || "Component";
}

// Usage: wrap the component
function StatusBar({ isOnline }) {
  return <div>{isOnline ? "● Online" : "○ Offline"}</div>;
}
const StatusBarWithOnlineStatus = withOnlineStatus(StatusBar);
// StatusBar now receives `isOnline` without knowing where it comes from
```

### HOC with Configuration

```javascript
// Factory HOC: takes config, returns a HOC
function withPermission(requiredRole) {
  return function (WrappedComponent) {
    function WithPermission({ userRole, ...props }) {
      if (userRole !== requiredRole) {
        return <AccessDenied />;
      }
      return <WrappedComponent {...props} />;
    }
    WithPermission.displayName = `WithPermission(${getDisplayName(WrappedComponent)}, ${requiredRole})`;
    return WithPermission;
  };
}

// Usage with configuration:
const AdminPanel = withPermission("admin")(DashboardPanel);
const ModeratorFeed = withPermission("moderator")(ContentFeed);
```

### TypeScript Typing for HOCs

```typescript
// Correctly typed HOC: separates injected props from passed-through props
function withLogger<P extends object>(WrappedComponent: ComponentType<P>) {
  function WithLogger({ ...props }: P) {
    useEffect(() => {
      console.log(`${getDisplayName(WrappedComponent)} rendered`);
    });
    return <WrappedComponent {...(props as P)} />;
  }

  WithLogger.displayName = `WithLogger(${getDisplayName(WrappedComponent)})`;
  return WithLogger;
}

// Injected-props HOC: the returned component has FEWER required props
interface WithAuthProps {
  user: User;
}

function withAuth<P extends WithAuthProps>(WrappedComponent: ComponentType<P>) {
  type OuterProps = Omit<P, keyof WithAuthProps>; // user is injected, not required externally

  function WithAuth(props: OuterProps) {
    const { user } = useAuthContext();
    if (!user) return <Redirect to="/login" />;
    return <WrappedComponent {...(props as P)} user={user} />;
  }

  WithAuth.displayName = `WithAuth(${getDisplayName(WrappedComponent)})`;
  return WithAuth;
}

// Usage: UserProfile doesn't need user as an external prop
const AuthenticatedProfile = withAuth(UserProfile);
<AuthenticatedProfile userId="42" /> // no `user` prop needed here
```

---

## 3. HOC Composition and Naming

```javascript
// HOCs compose by nesting function calls
const EnhancedComponent = compose(
  withAuth,
  withPermission("admin"),
  withLogger,
)(BaseComponent);

// Equivalent to:
const EnhancedComponent = withAuth(
  withPermission("admin")(withLogger(BaseComponent)),
);

// compose utility (right-to-left application):
function compose(...fns) {
  return (x) => fns.reduceRight((v, f) => f(v), x);
}

// Composition order matters:
// withAuth wraps withPermission wraps withLogger wraps BaseComponent
// Auth check happens outermost: if not authed, never reaches permission check
// Good pattern: put guards (auth, permission) outer, utilities (logger, analytics) inner
```

---

## 4. HOC Pitfalls

### Static Methods Are Dropped

```javascript
// ❌ Static methods from the wrapped component aren't on the HOC
class MyComponent extends React.Component {
  static someStaticMethod() {
    return "value";
  }
  render() {
    return null;
  }
}

const Enhanced = withHOC(MyComponent);
Enhanced.someStaticMethod; // undefined! HOC doesn't inherit statics

// ✅ Manually hoist statics using hoist-non-react-statics library
import hoistNonReactStatics from "hoist-non-react-statics";

function withHOC(WrappedComponent) {
  function WithHOC(props) {
    /* ... */
  }
  hoistNonReactStatics(WithHOC, WrappedComponent); // copies static methods
  return WithHOC;
}
```

### Ref Forwarding

```javascript
// ❌ Refs don't automatically pass through HOCs
const EnhancedInput = withHOC(Input);
const ref = useRef();
<EnhancedInput ref={ref} />; // ref points to WithHOC component, not Input

// ✅ forwardRef in the HOC
function withHOC(WrappedComponent) {
  const WithHOC = React.forwardRef((props, ref) => (
    <WrappedComponent {...props} ref={ref} />
  ));
  WithHOC.displayName = `WithHOC(${getDisplayName(WrappedComponent)})`;
  return WithHOC;
}
```

### Prop Name Collisions

```javascript
// ❌ HOC-injected prop name collides with caller's prop name
function withUser(WrappedComponent) {
  return (props) => {
    const user = getCurrentUser();
    return <WrappedComponent user={user} {...props} />; // spread order matters!
  };
}

// If caller passes user={anotherUser}: which wins depends on spread order
<UserCard user={anotherUser} />; // withUser injects user, then caller passes user

// ✅ Document injected prop names; provide a way to rename if needed
// Or: use a less likely-to-conflict namespace like _injectedUser
```

---

## 5. Render Props

A render prop passes a function as a prop (or as `children`); the component calls that function to delegate rendering:

```jsx
// Render prop: children as a function
function MouseTracker({ children }) {
  const [position, setPosition] = React.useState({ x: 0, y: 0 });

  function handleMouseMove(e) {
    setPosition({ x: e.clientX, y: e.clientY });
  }

  return (
    <div onMouseMove={handleMouseMove} style={{ height: "100vh" }}>
      {children(position)} // call children as a function, pass data in
    </div>
  );
}

// Usage: caller controls what renders with the mouse position
<MouseTracker>
  {({ x, y }) => (
    <div>
      <p>
        Mouse at: {x}, {y}
      </p>
      <img
        style={{ transform: `translate(${x}px, ${y}px)` }}
        src="/cursor.png"
      />
    </div>
  )}
</MouseTracker>;
```

### Named Render Prop (not children)

```jsx
// `render` prop is the original pattern; `children` is more common now
function DataFetcher({ url, render }) {
  const [data, setData] = React.useState(null);
  const [loading, setLoading] = React.useState(true);

  React.useEffect(() => {
    fetch(url)
      .then((r) => r.json())
      .then((data) => {
        setData(data);
        setLoading(false);
      });
  }, [url]);

  return render({ data, loading });
}

<DataFetcher
  url="/api/users"
  render={({ data, loading }) =>
    loading ? <Spinner /> : <UserList users={data} />
  }
/>;
```

### Compound Render Prop Pattern

```jsx
// Passing multiple render functions for complex slot arrangements
function DataTable({ columns, data, renderHeader, renderRow, renderEmpty }) {
  if (!data.length) return renderEmpty();

  return (
    <table>
      <thead>
        <tr>{columns.map((col) => renderHeader(col))}</tr>
      </thead>
      <tbody>
        {data.map((row, i) => (
          <tr key={i}>{columns.map((col) => renderRow(row, col))}</tr>
        ))}
      </tbody>
    </table>
  );
}

<DataTable
  columns={["name", "email", "role"]}
  data={users}
  renderHeader={(col) => <th>{col}</th>}
  renderRow={(user, col) => <td>{user[col]}</td>}
  renderEmpty={() => <EmptyState message="No users found" />}
/>;
```

---

## 6. Render Props vs Custom Hooks

```
CUSTOM HOOKS:

  Pros:
  ✓ No wrapper components — component tree stays flat
  ✓ Composable directly as function calls
  ✓ Multiple hooks can be combined in one component trivially
  ✓ No "wrapper hell" with 6 layers of context
  ✓ Better TypeScript: return types are just function return types
  ✓ DevTools: component tree is cleaner

  Cons:
  ✗ Only usable in function components
  ✗ Can't render anything — purely logic extraction

RENDER PROPS:

  Pros:
  ✓ Can be used in class components
  ✓ The calling code CONTROLS the JSX structure built around the shared state
  ✓ Enables some use cases hooks can't express: where the shared state
    needs to be positioned INSIDE specific JSX the caller provides

  Cons:
  ✗ Adds wrapper components to the tree
  ✗ Closure over render can cause performance issues (new function per render)
  ✗ "Wrapper hell" with many nested render props

WHEN TO PREFER RENDER PROPS OVER HOOKS:
  When the caller needs to control WHERE in JSX structure the output from
  shared state appears, and that positioning is non-trivial. Example:
  an animation library where the easing curves must be applied TO specific
  JSX nodes that the caller decides — hooks would require the caller to
  manually thread values through JSX.

  In practice: custom hooks cover ~95% of render prop use cases more cleanly.
```

---

## 7. The Wrapper Hell Problem

```
WRAPPER HELL: component tree buried under many HOC/render prop wrappers

In React DevTools, a component with many HOCs looks like:
  WithAuth
    WithPermission
      WithLogger
        WithTheme
          WithAnalytics
            WithStyles
              ActualComponent  ← the actual component, 6 levels deep!

Consequences:
  - DevTools is hard to read and debug
  - Error messages mention wrapper names, not the actual component
  - Performance: many extra component instances to render
  - Props: hard to trace where a given prop came from

HOOKS SOLUTION:
  function ActualComponent(props) {
    useAuth();           // flat — no wrapper
    usePermission();     // flat
    useLogger();         // flat
    useTheme();          // flat
    useAnalytics();      // flat
    useStyles();         // flat
    // ... component code
  }
  // DevTools: just ActualComponent, all hooks visible inline
```

---

## 8. When HOCs Still Make Sense

```
1. LIBRARY INTEGRATION WITH CLASS COMPONENT CODEBASES:
   Redux `connect()`, MobX `observer()`, Relay GraphQL `createFragmentContainer()`
   These ship as HOCs and wrap class components. If your codebase uses class
   components, HOCs are the mechanism.

2. CROSS-CUTTING CONCERNS APPLIED TO ENTIRE COMPONENT FAMILIES:
   If you need to ensure ALL form components receive specific props from context,
   a HOC applied via a shared factory function is a natural solution.

3. CODE SPLITTING WITH LAZY LOADING:
   React.lazy() is conceptually a HOC that wraps a component with deferred loading.

4. ROUTE-LEVEL GUARDS IN OLDER CODEBASES:
   <Route component={withAuth(Dashboard)} /> — HOC applied at routing level

5. ERROR BOUNDARIES:
   Error boundaries must be class components; wrapping them as HOCs is
   the ergonomic way to apply them.

   function withErrorBoundary(Component, FallbackComponent) {
     return class WithErrorBoundary extends React.Component {
       state = { hasError: false };
       static getDerivedStateFromError() { return { hasError: true }; }
       render() {
         if (this.state.hasError) return <FallbackComponent />;
         return <Component {...this.props} />;
       }
     };
   }
```

---

## 9. When Render Props Still Make Sense

```jsx
// 1. CONSUMER NEEDS CONTROL OVER STRUCTURE AROUND SHARED STATE

// Example: virtualized list where the render function for each row
// must be provided by the consumer and positioned exactly
function VirtualizedList({ items, rowHeight, renderRow }) {
  const [scrollTop, setScrollTop] = useState(0);
  const visibleStart = Math.floor(scrollTop / rowHeight);
  const visibleEnd   = visibleStart + Math.ceil(windowHeight / rowHeight);

  return (
    <div style={{ height: items.length * rowHeight }} onScroll={...}>
      {items.slice(visibleStart, visibleEnd).map((item, i) =>
        <div key={item.id} style={{ position: 'absolute', top: (visibleStart + i) * rowHeight }}>
          {renderRow(item)} // consumer decides what each row looks like
        </div>
      )}
    </div>
  );
}
// The consumer's renderRow is positioned INSIDE complex virtualization logic

// 2. DOWNWARD DATA FLOW INTO SPECIFIC JSX NODES (unusual but valid)
// Animation libraries like react-spring use render props in some cases
// to thread spring values into specific JSX positions the library can't
// know about in advance.

// 3. HEADLESS UI COMPONENTS (Radix, Headless UI)
// Some headless component libraries use render props for their most flexible
// composition patterns, allowing callers to provide the DOM structure.
```

---

## 10. Migrating from HOC/Render Props to Hooks

```jsx
// BEFORE: HOC pattern
function withMousePosition(WrappedComponent) {
  function WithMousePosition(props) {
    const [position, setPosition] = useState({ x: 0, y: 0 });
    useEffect(() => {
      function handler(e) {
        setPosition({ x: e.clientX, y: e.clientY });
      }
      window.addEventListener("mousemove", handler);
      return () => window.removeEventListener("mousemove", handler);
    }, []);
    return <WrappedComponent {...props} mousePosition={position} />;
  }
  return WithMousePosition;
}

const ComponentWithMouse = withMousePosition(MyComponent);

// AFTER: Custom hook (same logic, no wrapper component)
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  useEffect(() => {
    function handler(e) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
    window.addEventListener("mousemove", handler);
    return () => window.removeEventListener("mousemove", handler);
  }, []);
  return position;
}

// Component uses the hook directly — no HOC needed
function MyComponent() {
  const mousePosition = useMousePosition(); // same behavior, simpler structure
  return (
    <div>
      Mouse: {mousePosition.x}, {mousePosition.y}
    </div>
  );
}
```

```jsx
// BEFORE: Render prop pattern
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  // ... handler setup ...
  return <div onMouseMove={...}>{children(pos)}</div>;
}

<MouseTracker>
  {pos => <span>Mouse: {pos.x}, {pos.y}</span>}
</MouseTracker>

// AFTER: Custom hook
function MyComponent() {
  const pos = useMousePosition();
  return <div><span>Mouse: {pos.x}, {pos.y}</span></div>;
  // No wrapper div from MouseTracker — the caller's div serves both purposes
}
```

---

## 11. Good Practices

### ✅ Always set displayName on HOCs

```javascript
// ✅ DevTools shows "WithAuth(Dashboard)" instead of "WithAuth"
WithAuth.displayName = `WithAuth(${getDisplayName(WrappedComponent)})`;
```

### ✅ Use forwardRef in HOCs when wrapping input-like components

```javascript
// ✅ Refs must be explicitly forwarded through HOCs
const WithHOC = React.forwardRef((props, ref) => (
  <WrappedComponent {...props} ref={ref} />
));
```

### ✅ Default to custom hooks over HOCs for new code

```
✅ New feature: write a custom hook first
   If you find yourself creating a HOC: ask if a custom hook would serve
   the same purpose more simply (it almost always will for function components)
```

---

## 12. Bad Practices

### ❌ Creating HOCs inside render functions

```jsx
// ❌ New HOC created every render — wrapped component loses identity, unmounts/remounts
function Parent({ condition }) {
  const EnhancedChild = withHOC(Child); // defined inside the component! recreated every render
  return <EnhancedChild />;
}

// ✅ HOC applied outside the component, once
const EnhancedChild = withHOC(Child);
function Parent({ condition }) {
  return <EnhancedChild />;
}
```

### ❌ HOC chains where hooks would be simpler

```javascript
// ❌ Three nested HOCs just to inject data
const Component = withAuth(withData(withAnalytics(BaseComponent)));

// ✅ Three hooks in the component
function Component() {
  const { user } = useAuth();
  const { data } = useData();
  const { track } = useAnalytics();
  // ...
}
```

---

## 13. Common Mistakes

### Mistake 1 — Not passing unknown props through HOCs

```javascript
// ❌ HOC that consumes props but doesn't forward the rest
function withUser(WrappedComponent) {
  function WithUser({ userId }) {
    const user = getUser(userId);
    return <WrappedComponent user={user} />; // ❌ only passes `user`, drops all other props!
  }
  return WithUser;
}

// ✅ Spread through all non-injected props
function withUser(WrappedComponent) {
  function WithUser({ userId, ...rest }) {
    const user = getUser(userId);
    return <WrappedComponent {...rest} user={user} />; // ✅ forwards everything else
  }
  return WithUser;
}
```

### Mistake 2 — Performance issue: new function per render in render props

```jsx
// ❌ New function per render causes re-renders of the consumer
class Parent extends React.Component {
  render() {
    return (
      <DataProvider
        render={({ data }) => <Consumer data={data} />} // new function every render!
      />
    );
  }
}

// ✅ Extract to class method (class component) or memoize (function component)
class Parent extends React.Component {
  renderConsumer = ({ data }) => <Consumer data={data} />; // stable reference
  render() {
    return <DataProvider render={this.renderConsumer} />;
  }
}
```

---

## 14. Interview-Level Explanation

> **"What are HOCs and render props? Why did hooks largely replace them?"**

**Strong answer:**

> "Both HOCs and render props are patterns for sharing stateful logic between React components — extracting logic that would otherwise be duplicated into a reusable abstraction.
>
> A HOC is a function that takes a component and returns a new, enhanced component with additional behavior or injected props. `connect()` from react-redux is the canonical example — it wraps your component in a component that subscribes to the Redux store and injects selected state as props. HOCs are pure function composition: `const Enhanced = withAuth(withPermission('admin')(MyComponent))`. The pattern works well but has structural costs — each HOC adds a wrapper component to the tree, making DevTools harder to read, making error messages confusing (they reference `WithAuth` instead of `MyComponent`), and causing prop name collision risks when multiple HOCs inject props with the same name.
>
> Render props invert the control: instead of the HOC deciding what to render, the component calls a function prop — often `children` — passing its state as arguments, and the caller decides what to render with that state. This gives the caller more control over where in JSX the shared state is used, at the cost of adding wrapper components and creating a closure-heavy nesting pattern that people called 'wrapper hell.'
>
> Custom hooks replaced both patterns for most use cases. A hook extracts the same stateful logic into a function, but returns values instead of rendering anything — no wrapper component, no prop injection, no tree pollution. The component calls the hook like any function, receives the values, and uses them however it wants. The mental model is simpler, composition is easier (you can call multiple hooks in one component without nesting), and the component tree stays flat.
>
> HOCs remain relevant for three specific cases: integrating with library APIs that ship as HOCs (react-redux `connect`, MobX observer), error boundaries which must be class components and are ergonomically wrapped as HOCs, and applying cross-cutting concerns across a component family. Render props still appear in headless component libraries like Radix UI for their most flexible composition modes, and in virtualized list implementations where the consumer must provide a render function for items that gets positioned inside complex internal logic."

---

## 15. Exercises

### Exercise 1 — Convert a HOC to a custom hook

```jsx
// Convert this HOC to an equivalent custom hook + usage example

function withWindowSize(WrappedComponent) {
  function WithWindowSize(props) {
    const [size, setSize] = React.useState({
      width: window.innerWidth,
      height: window.innerHeight,
    });

    React.useEffect(() => {
      function handleResize() {
        setSize({ width: window.innerWidth, height: window.innerHeight });
      }
      window.addEventListener("resize", handleResize);
      return () => window.removeEventListener("resize", handleResize);
    }, []);

    return (
      <WrappedComponent
        {...props}
        windowWidth={size.width}
        windowHeight={size.height}
      />
    );
  }
  WithWindowSize.displayName = `WithWindowSize(${WrappedComponent.displayName || WrappedComponent.name})`;
  return WithWindowSize;
}

// Original usage:
const ResponsiveComponent = withWindowSize(MyComponent);
// <ResponsiveComponent /> — MyComponent receives windowWidth, windowHeight
```

<details>
<summary>Solution</summary>

```tsx
// Custom hook: same logic, no wrapper component
function useWindowSize() {
  const [size, setSize] = React.useState({
    width: typeof window !== "undefined" ? window.innerWidth : 0,
    height: typeof window !== "undefined" ? window.innerHeight : 0,
  });

  React.useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return size;
}

// Updated component: uses hook directly, no HOC needed
function MyComponent() {
  const { width, height } = useWindowSize(); // replaces windowWidth/windowHeight props

  return (
    <div>
      Window: {width} × {height}
      {width < 768 ? <MobileLayout /> : <DesktopLayout />}
    </div>
  );
}

// Benefits of the migration:
// 1. No wrapper component in DevTools (flat tree)
// 2. The component owns the variable names (not constrained to windowWidth/windowHeight)
// 3. Testable by mocking window.innerWidth without prop injection
// 4. Multiple hooks composable without nesting
// 5. SSR-safe: typeof window check in the hook itself

// If HOC compatibility is needed for class components or library consumers:
// the HOC can be thin wrapper around the hook:
function withWindowSize(WrappedComponent) {
  function WithWindowSize(props) {
    const { width, height } = useWindowSize(); // hook inside HOC
    return (
      <WrappedComponent {...props} windowWidth={width} windowHeight={height} />
    );
  }
  WithWindowSize.displayName = `WithWindowSize(${getDisplayName(WrappedComponent)})`;
  return WithWindowSize;
}
```

</details>

---

## 🔗 Related Topics

- [`patterns/02-custom-hooks.md`](./02-custom-hooks.md) — The modern replacement for most HOC/render prop use cases
- [`patterns/01-component-composition.md`](./01-component-composition.md) — UI-level composition
- [`patterns/05-compound-components.md`](./05-compound-components.md) — Advanced compositional patterns
- [`javascript-core/04-closures.md`](../javascript-core/04-closures.md) — Closures underpinning render props

---

<div align="center">

**Next:** [`patterns/04-controlled-uncontrolled.md`](./04-controlled-uncontrolled.md) →

</div>
