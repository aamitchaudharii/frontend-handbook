# 01 — Component Composition

> **"Composition is the answer to the question inheritance can't solve cleanly: how do components share behavior and structure without becoming rigid, tightly coupled hierarchies? React's entire component model is a bet on composition over inheritance — and understanding why that bet pays off is foundational to writing components that scale."**

Component composition is the practice of building complex UI by combining small, focused components rather than building monolithic ones or relying on inheritance hierarchies. React was explicitly designed around composition — `props.children`, render props, and component slots are all composition mechanisms. This document covers the core composition patterns, when each applies, and the architectural reasoning that makes composition superior to inheritance for UI code.

---

## 📚 Table of Contents

1. [Why Composition Over Inheritance](#1-why-composition-over-inheritance)
2. [children as the Primary Composition Tool](#2-children-as-the-primary-composition-tool)
3. [Slot-Based Composition](#3-slot-based-composition)
4. [Specialization Through Composition](#4-specialization-through-composition)
5. [Containment Patterns](#5-containment-patterns)
6. [Composition with TypeScript](#6-composition-with-typescript)
7. [Polymorphic Components (as prop)](#7-polymorphic-components-as-prop)
8. [Composing Behavior vs Composing UI](#8-composing-behavior-vs-composing-ui)
9. [Avoiding Prop Drilling Through Composition](#9-avoiding-prop-drilling-through-composition)
10. [When Composition Isn't Enough](#10-when-composition-isnt-enough)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. Why Composition Over Inheritance

```jsx
// ❌ Inheritance approach (not idiomatic React — shown for contrast)
class Button extends Component {
  renderIcon() {
    return null;
  }
  render() {
    return (
      <button className="btn">
        {this.renderIcon()}
        {this.props.children}
      </button>
    );
  }
}

class IconButton extends Button {
  renderIcon() {
    return <Icon name={this.props.iconName} />;
  }
}
// Problems:
// - Tight coupling: IconButton depends on Button's internal renderIcon() contract
// - Rigid hierarchy: what if you need a button with BOTH an icon AND a badge?
//   Multiple inheritance doesn't exist in JS — you'd need ButtonWithIconAndBadge
// - Hard to test in isolation: IconButton can't be tested without Button's full behavior
// - The "fragile base class" problem: changing Button can break all subclasses unexpectedly
```

```jsx
// ✅ Composition approach (idiomatic React)
function Button({ children, icon, ...props }) {
  return (
    <button className="btn" {...props}>
      {icon}
      {children}
    </button>
  );
}

// Usage: compose exactly what's needed, no rigid hierarchy
<Button icon={<Icon name="star" />}>Favorite</Button>
<Button icon={<Icon name="trash" />}>Delete</Button>
<Button>Plain button, no icon</Button>

// Need icon AND badge? Just compose more:
<Button icon={<Icon name="bell" />}>
  Notifications <Badge count={5} />
</Button>
// No class hierarchy needed — components combine like building blocks
```

### The Core Insight

```
INHERITANCE asks: "What IS this component?" (IconButton IS-A Button)
COMPOSITION asks: "What does this component HAVE or USE?" (Button HAS an icon slot)

Composition produces a flat, flexible structure:
  Button + icon prop + children = IconButton-like behavior, no subclass needed
  Button + onClick + loading state = LoadingButton-like behavior, no subclass needed

Any combination is achievable without combinatorial explosion of subclasses.
```

---

## 2. children as the Primary Composition Tool

```jsx
// The simplest and most powerful composition primitive
function Card({ children }) {
  return <div className="card">{children}</div>;
}

function Panel({ children }) {
  return (
    <Card>
      <div className="panel-header">Settings</div>
      <div className="panel-body">{children}</div>
    </Card>
  );
}

// Components compose naturally through nesting
<Panel>
  <UserSettingsForm />
  <NotificationPreferences />
</Panel>;
```

### children as a Function (Render Prop via children)

```jsx
// children can be a function for cases needing data flow back to the parent's children
function Toggle({ children }) {
  const [isOn, setOn] = useState(false);
  return children({ isOn, toggle: () => setOn((o) => !o) });
}

// Usage: caller controls the rendering, Toggle controls the logic
<Toggle>
  {({ isOn, toggle }) => (
    <button onClick={toggle}>{isOn ? "ON" : "OFF"}</button>
  )}
</Toggle>;

// This pattern has mostly been superseded by custom hooks (see Section 8),
// but remains useful when JSX structure itself needs to vary based on state
```

---

## 3. Slot-Based Composition

When a component needs multiple distinct content areas, named props act as "slots":

```jsx
// Multiple named slots via props
function PageLayout({ header, sidebar, children, footer }) {
  return (
    <div className="page-layout">
      <header className="page-header">{header}</header>
      <div className="page-body">
        <aside className="page-sidebar">{sidebar}</aside>
        <main className="page-content">{children}</main>
      </div>
      <footer className="page-footer">{footer}</footer>
    </div>
  );
}

// Usage: each slot receives exactly the content it needs
<PageLayout
  header={<TopNav />}
  sidebar={<FilterPanel />}
  footer={<SiteFooter />}
>
  <ProductGrid />
</PageLayout>;
```

### Slot Pattern with Sub-Components

```jsx
// Sub-components attached as properties for discoverable slot APIs
function Modal({ children }) {
  return <div className="modal">{children}</div>;
}
Modal.Header = function ModalHeader({ children }) {
  return <div className="modal-header">{children}</div>;
};
Modal.Body = function ModalBody({ children }) {
  return <div className="modal-body">{children}</div>;
};
Modal.Footer = function ModalFooter({ children }) {
  return <div className="modal-footer">{children}</div>;
};

// Usage: reads like a structured document, IDE autocomplete shows Modal.* options
<Modal>
  <Modal.Header>Confirm Deletion</Modal.Header>
  <Modal.Body>This action cannot be undone.</Modal.Body>
  <Modal.Footer>
    <Button variant="ghost">Cancel</Button>
    <Button variant="danger">Delete</Button>
  </Modal.Footer>
</Modal>;
```

---

## 4. Specialization Through Composition

Building specialized components by composing a generic base:

```jsx
// Generic base component
function Dialog({ title, children, actions, onClose }) {
  return (
    <div className="dialog">
      <div className="dialog-header">
        <h2>{title}</h2>
        <button onClick={onClose}>×</button>
      </div>
      <div className="dialog-body">{children}</div>
      <div className="dialog-actions">{actions}</div>
    </div>
  );
}

// Specialized: ConfirmDialog composes Dialog, doesn't extend it
function ConfirmDialog({ message, onConfirm, onCancel }) {
  return (
    <Dialog
      title="Confirm"
      onClose={onCancel}
      actions={
        <>
          <Button onClick={onCancel} variant="ghost">
            Cancel
          </Button>
          <Button onClick={onConfirm} variant="primary">
            Confirm
          </Button>
        </>
      }
    >
      <p>{message}</p>
    </Dialog>
  );
}

// Specialized: AlertDialog composes Dialog differently
function AlertDialog({ message, onAcknowledge }) {
  return (
    <Dialog
      title="Alert"
      onClose={onAcknowledge}
      actions={<Button onClick={onAcknowledge}>OK</Button>}
    >
      <p>{message}</p>
    </Dialog>
  );
}

// Both ConfirmDialog and AlertDialog are independent, testable, and don't
// share a class hierarchy — they share Dialog by composing it
```

---

## 5. Containment Patterns

### Wrapper Components

```jsx
// A wrapper adds behavior/styling without knowing about its children's internals
function ScrollableContainer({ children, maxHeight = "400px" }) {
  return (
    <div className="scrollable" style={{ maxHeight, overflowY: "auto" }}>
      {children}
    </div>
  );
}

function ErrorBoundaryWrapper({ children, fallback }) {
  return <ErrorBoundary fallback={fallback}>{children}</ErrorBoundary>;
}

// Stacking wrappers — each adds one concern, none know about the others
<ErrorBoundaryWrapper fallback={<ErrorMessage />}>
  <ScrollableContainer maxHeight="600px">
    <Suspense fallback={<Spinner />}>
      <ProductList />
    </Suspense>
  </ScrollableContainer>
</ErrorBoundaryWrapper>;
```

### Provider Composition

```jsx
// Multiple context providers compose via nesting
function AppProviders({ children }) {
  return (
    <ThemeProvider>
      <AuthProvider>
        <QueryClientProvider client={queryClient}>
          <ToastProvider>{children}</ToastProvider>
        </QueryClientProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

// Cleaner: a composeProviders utility flattens the nesting
function composeProviders(...providers) {
  return ({ children }) =>
    providers.reduceRight(
      (acc, Provider) => <Provider>{acc}</Provider>,
      children,
    );
}

const AppProviders = composeProviders(
  ThemeProvider,
  AuthProvider,
  (props) => <QueryClientProvider client={queryClient} {...props} />,
  ToastProvider,
);
```

---

## 6. Composition with TypeScript

```typescript
// Typed children slots
interface CardProps {
  children: ReactNode;
}

interface PageLayoutProps {
  header:  ReactNode;
  sidebar?: ReactNode; // optional slot
  footer:  ReactNode;
  children: ReactNode;
}

function PageLayout({ header, sidebar, footer, children }: PageLayoutProps) {
  return (
    <div>
      <header>{header}</header>
      {sidebar && <aside>{sidebar}</aside>}
      <main>{children}</main>
      <footer>{footer}</footer>
    </div>
  );
}

// Typed sub-component pattern
interface ModalComponent extends FC<{ children: ReactNode }> {
  Header: FC<{ children: ReactNode }>;
  Body:   FC<{ children: ReactNode }>;
  Footer: FC<{ children: ReactNode }>;
}

const Modal: ModalComponent = ({ children }) => (
  <div className="modal">{children}</div>
);
Modal.Header = ({ children }) => <div className="modal-header">{children}</div>;
Modal.Body   = ({ children }) => <div className="modal-body">{children}</div>;
Modal.Footer = ({ children }) => <div className="modal-footer">{children}</div>;

// Type-safe: Modal.Header, Modal.Body, Modal.Footer are all recognized by TS
```

---

## 7. Polymorphic Components (as prop)

A component that can render as different underlying HTML elements while preserving its styling and behavior:

```typescript
// Polymorphic component: render as any element/component via `as` prop
type PolymorphicProps<E extends ElementType> = {
  as?: E;
  children?: ReactNode;
} & ComponentPropsWithoutRef<E>;

function Text<E extends ElementType = 'span'>({
  as,
  children,
  ...props
}: PolymorphicProps<E>) {
  const Component = as || 'span';
  return <Component {...props}>{children}</Component>;
}

// Usage: same component, different underlying elements
<Text>Default span</Text>
<Text as="h1" className="title">Heading</Text>
<Text as="label" htmlFor="email">Email</Text>
<Text as={Link} to="/profile">Profile Link</Text> {/* even custom components */}

// Real-world use case: design system Button that can render as <a> for links
function Button<E extends ElementType = 'button'>({
  as,
  variant = 'primary',
  ...props
}: PolymorphicProps<E> & { variant?: 'primary' | 'secondary' }) {
  const Component = as || 'button';
  return <Component className={`btn btn--${variant}`} {...props} />;
}

<Button onClick={handleSave}>Save</Button>              {/* renders <button> */}
<Button as="a" href="/docs">Documentation</Button>       {/* renders <a> */}
<Button as={RouterLink} to="/settings">Settings</Button> {/* renders router Link */}
```

---

## 8. Composing Behavior vs Composing UI

```jsx
// UI composition: combining visual structure (covered above)

// BEHAVIOR composition: custom hooks compose LOGIC the same way components compose UI
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debounced;
}

// Composing multiple hooks = composing multiple pieces of behavior
function SearchBox() {
  const [query, setQuery]     = useState('');
  const debouncedQuery        = useDebounce(query, 300);
  const [isOpen, toggleOpen]  = useToggle(false);
  const { data: results }     = useQuery({
    queryKey: ['search', debouncedQuery],
    queryFn:  () => searchApi(debouncedQuery),
    enabled:  debouncedQuery.length > 2,
  });

  // Three independent, reusable behaviors composed together —
  // none of these hooks know about each other
  return (/* JSX using query, isOpen, results */);
}
```

---

## 9. Avoiding Prop Drilling Through Composition

Composition itself can solve prop drilling without reaching for Context:

```jsx
// ❌ Prop drilling: passing data through intermediate components that don't use it
function App() {
  const [user, setUser] = useState(null);
  return <Layout user={user} />;
}
function Layout({ user }) {
  return <Sidebar user={user} />; // Layout doesn't use `user`, just forwards it
}
function Sidebar({ user }) {
  return <UserProfile user={user} />; // Sidebar doesn't use `user` either
}
function UserProfile({ user }) {
  return <p>{user.name}</p>; // finally used here
}

// ✅ Composition: pass the FULLY RENDERED component down, skip the drilling
function App() {
  const [user, setUser] = useState(null);
  return <Layout sidebarContent={<UserProfile user={user} />} />;
}
function Layout({ sidebarContent }) {
  return <Sidebar content={sidebarContent} />; // passes the element through, doesn't need `user`
}
function Sidebar({ content }) {
  return <div className="sidebar">{content}</div>;
}
// UserProfile is constructed once, at the top, where `user` is in scope
// Layout and Sidebar never need to know `user` exists
```

```jsx
// children-based version: even simpler when nesting allows it
function App() {
  const [user, setUser] = useState(null);
  return (
    <Layout>
      <Sidebar>
        <UserProfile user={user} />
      </Sidebar>
    </Layout>
  );
}
function Layout({ children }) {
  return <div className="layout">{children}</div>;
}
function Sidebar({ children }) {
  return <aside>{children}</aside>;
}
// Zero prop drilling — UserProfile is composed directly where `user` is available
```

---

## 10. When Composition Isn't Enough

```
COMPOSITION SOLVES:
  ✓ Avoiding rigid component hierarchies
  ✓ Sharing UI structure flexibly
  ✓ Avoiding prop drilling for STRUCTURAL data (rendered elements)
  ✓ Building specialized components from generic ones

COMPOSITION DOES NOT SOLVE:
  ✗ Sharing STATE across deeply nested, non-adjacent components
    → Use Context, or a state management library
  ✗ Cross-cutting concerns that need to wrap MANY unrelated components
    → Use Context + custom hook, or a HOC for legacy codebases
  ✗ Global app state (auth, theme, feature flags)
    → Use Context or external store (Zustand, Redux)
  ✗ Imperative actions across component boundaries (focus management, etc.)
    → Use refs + forwardRef, or imperative handles

RULE OF THUMB:
  If you're passing data DOWN through component boundaries that don't use it →
    refactor with composition (pass the rendered element instead)
  If many SEPARATE, UNRELATED parts of the tree need the SAME data →
    use Context (composition can't reach across separate branches)
```

---

## 11. Good Practices

### ✅ Default to children for single content areas

```jsx
// ✅ Simple, idiomatic
function Card({ children }) {
  return <div className="card">{children}</div>;
}
```

### ✅ Use named slots when multiple distinct content areas exist

```jsx
// ✅ Clear intent for multi-area layouts
function Layout({ header, sidebar, children }) {
  /* ... */
}
```

### ✅ Compose specialized components from generic bases

```jsx
// ✅ ConfirmDialog composes Dialog rather than extending a base class
function ConfirmDialog(props) {
  return <Dialog {...buildConfirmProps(props)} />;
}
```

---

## 12. Bad Practices

### ❌ Deeply nested prop drilling instead of composition

```jsx
// ❌ Passing the same prop through 5 levels just to reach the bottom
<A data={data}><B data={data}><C data={data}><D data={data} /></C></B></A>
// ✅ Compose the consumer where data is available, pass the element down
<A><B><C>{renderD(data)}</C></B></A>
```

### ❌ Over-engineering with HOCs when composition suffices

```jsx
// ❌ HOC wrapping for something composition handles more simply
const withCard = (Component) => (props) => (
  <div className="card">
    <Component {...props} />
  </div>
);
const EnhancedProfile = withCard(Profile);

// ✅ Just compose directly
<Card>
  <Profile {...props} />
</Card>;
```

---

## 13. Common Mistakes

### Mistake 1 — Confusing composition with prop spreading

```jsx
// ❌ Spreading everything isn't composition — it's just prop forwarding
function Wrapper(props) {
  return <Inner {...props} />; // not really composing, just passing through
}

// ✅ True composition involves DECIDING what to combine, not blindly forwarding
function Wrapper({ children, ...rest }) {
  return (
    <div className="wrapper" {...rest}>
      <Header />
      {children}
      <Footer />
    </div>
  );
}
```

### Mistake 2 — Breaking composition by requiring specific child types

```jsx
// ❌ Tightly coupling parent to specific child component types
function Tabs({ children }) {
  return children.map((child) => {
    if (child.type !== Tab) throw new Error("Tabs only accepts Tab children");
    return child;
  });
}
// Brittle — breaks if children include Fragment, conditional null, etc.

// ✅ Use a more flexible API: array of data + render function, or React.Children utilities
function Tabs({ items, renderTab }) {
  return items.map(renderTab);
}
```

---

## 14. Interview-Level Explanation

> **"Why does React favor composition over inheritance? How do you decide between children, named slots, and a render prop?"**

**Strong answer:**

> "React's documentation explicitly states they haven't found a use case where inheritance is recommended over composition, and the reasoning is structural. Inheritance creates a rigid hierarchy — a subclass IS-A superclass, locked into that relationship at definition time, and you can't easily combine behaviors from multiple unrelated base classes in JavaScript. Composition instead asks what a component HAS or USES, expressed through props — particularly `children` — which can be assembled in any combination at the call site rather than baked into a class hierarchy.
>
> For deciding which composition mechanism to use: `children` is the default and handles the vast majority of cases — anything with a single, primary content area. Named slot props — `header`, `sidebar`, `footer` — are appropriate when a component has multiple distinct content regions that aren't simply nested inside each other; this is common in layout components. The sub-component pattern, where you attach `Modal.Header`, `Modal.Body`, `Modal.Footer` as properties on the main component, gives you a discoverable, IDE-autocompletable API for structured content while still being pure composition under the hood — there's no class hierarchy, just components passed as children.
>
> Render props via `children` as a function are useful when the parent needs to expose state or behavior that the child's rendering depends on dynamically — though most of those use cases have been superseded by custom hooks, since hooks compose logic the same way components compose UI, without the JSX nesting overhead.
>
> One underappreciated benefit of composition: it solves a category of prop drilling that people often reach for Context to fix. If you're passing data through several layers of components that don't use it themselves, just to reach a deeply nested consumer, the fix is often not Context but restructuring so you construct the consumer where the data is available and pass the already-rendered element down as a prop or children. The intermediate components never need to know the data exists. Context is for genuinely separate branches of the tree needing the same data — composition can't reach across sibling branches, only down through ancestor-descendant relationships."

---

## 15. Exercises

### Exercise 1 — Refactor inheritance-style code to composition

```jsx
// Refactor this rigid component hierarchy into composition-based components

class BaseCard extends React.Component {
  renderHeader() {
    return null;
  }
  renderFooter() {
    return null;
  }
  render() {
    return (
      <div className="card">
        {this.renderHeader()}
        <div className="card-body">{this.props.children}</div>
        {this.renderFooter()}
      </div>
    );
  }
}

class ProductCard extends BaseCard {
  renderHeader() {
    return <div className="card-header">{this.props.title}</div>;
  }
  renderFooter() {
    return <button onClick={this.props.onAddToCart}>Add to Cart</button>;
  }
}
```

<details>
<summary>Solution</summary>

```jsx
// ✅ Composition-based: a generic Card with slots
function Card({ header, footer, children }) {
  return (
    <div className="card">
      {header && <div className="card-header">{header}</div>}
      <div className="card-body">{children}</div>
      {footer}
    </div>
  );
}

// ProductCard composes Card instead of extending it
function ProductCard({ title, onAddToCart, children }) {
  return (
    <Card
      header={title}
      footer={<button onClick={onAddToCart}>Add to Cart</button>}
    >
      {children}
    </Card>
  );
}

// Benefits over the inheritance version:
// - Card is independently usable WITHOUT any product-specific logic
// - ProductCard is a thin, testable function — no class lifecycle to manage
// - Adding a new card variant (e.g., ArticleCard) doesn't require modifying Card
// - Multiple "footer" combinations are trivial: just pass different elements
function ArticleCard({ title, author, onShare, children }) {
  return (
    <Card
      header={
        <>
          <h3>{title}</h3>
          <span className="author">{author}</span>
        </>
      }
      footer={<button onClick={onShare}>Share</button>}
    >
      {children}
    </Card>
  );
}
```

</details>

---

## 🔗 Related Topics

- [`patterns/02-custom-hooks.md`](./02-custom-hooks.md) — Composing behavior via hooks
- [`patterns/03-render-props-hoc.md`](./03-render-props-hoc.md) — Historical composition patterns
- [`patterns/05-compound-components.md`](./05-compound-components.md) — Advanced slot-based composition
- [`system-design/02-feature-based-structure.md`](../system-design/02-feature-based-structure.md) — Composition at the architecture level

---

<div align="center">

**Next:** [`patterns/02-custom-hooks.md`](./02-custom-hooks.md) →

</div>
