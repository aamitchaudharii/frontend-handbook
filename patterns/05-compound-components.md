# 05 — Compound Components

> **"The compound component pattern is the difference between a Select component that does everything (here are the options, here is the styling, here is the trigger, no you can't change any of it) and one that does the right things (I handle open/close, keyboard navigation, and ARIA — you handle what you render and where). The first is simpler to use once. The second is composable forever."**

The compound component pattern builds component APIs that share implicit state between a parent and its children, without prop drilling and without forcing a rigid pre-determined structure on the caller. The parent manages the shared state; the children access it implicitly via Context. The caller assembles the pieces however they need — reordering, omitting, adding custom elements between them — while the shared behavior (toggle state, keyboard handling, ARIA attributes) remains correct. This pattern is the foundation of headless component libraries like Radix UI, Headless UI, and Reach UI.

---

## 📚 Table of Contents

1. [What Compound Components Solve](#1-what-compound-components-solve)
2. [Basic Compound Component (Sub-Components via Properties)](#2-basic-compound-component-sub-components-via-properties)
3. [Context-Based Compound Components](#3-context-based-compound-components)
4. [Flexible Composition Without Rigid Structure](#4-flexible-composition-without-rigid-structure)
5. [Controlled vs Uncontrolled Compound Components](#5-controlled-vs-uncontrolled-compound-components)
6. [Keyboard Navigation and ARIA](#6-keyboard-navigation-and-aria)
7. [Headless Components](#7-headless-components)
8. [Compound Components with TypeScript](#8-compound-components-with-typescript)
9. [Real-World Example — Accordion](#9-real-world-example--accordion)
10. [Real-World Example — Tabs](#10-real-world-example--tabs)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. What Compound Components Solve

```jsx
// ❌ THE MONOLITHIC COMPONENT PROBLEM:
// All customization via props — the component controls everything
<Select
  options={items}
  placeholder="Choose one..."
  renderOption={(item) => <div>{item.label}</div>}
  renderSelected={(item) => <strong>{item.label}</strong>}
  onSelect={handleSelect}
  disabled={isLoading}
  isSearchable
  noOptionsMessage="Nothing found"
  classNamePrefix="my-select"
  menuPortalTarget={document.body}
  formatOptionLabel={...}
  // ... 30 more props to customize every pixel
/>
// Adding a new customization not covered: impossible without a new prop

// ✅ THE COMPOUND COMPONENT SOLUTION:
// The caller assembles the pieces; the parent handles shared state
<Select value={selected} onChange={setSelected}>
  <Select.Trigger>
    {selected ? selected.label : 'Choose one...'}
    <ChevronIcon />
  </Select.Trigger>

  <Select.Menu>
    {items.map(item => (
      <Select.Option key={item.id} value={item.id}>
        <ItemIcon name={item.icon} />   {/* custom icon per item */}
        {item.label}
        {item.isPremium && <Badge>Pro</Badge>} {/* conditional badge */}
      </Select.Option>
    ))}
  </Select.Menu>
</Select>
// Caller controls the structure; Select handles open/close, keyboard nav, ARIA
```

---

## 2. Basic Compound Component (Sub-Components via Properties)

The simplest form: sub-components attached as properties, no shared Context needed:

```jsx
// When sub-components don't need to share state — just structure
function Card({ children }) {
  return <div className="card">{children}</div>;
}

Card.Header = function CardHeader({ children, actions }) {
  return (
    <div className="card-header">
      <div className="card-header-content">{children}</div>
      {actions && <div className="card-header-actions">{actions}</div>}
    </div>
  );
};

Card.Body = function CardBody({ children, padded = true }) {
  return (
    <div className={`card-body ${padded ? "card-body--padded" : ""}`}>
      {children}
    </div>
  );
};

Card.Footer = function CardFooter({ children, align = "right" }) {
  return <div className={`card-footer card-footer--${align}`}>{children}</div>;
};

// Usage: naturally structured, IDE-discoverable sub-components
<Card>
  <Card.Header actions={<Button size="sm">Edit</Button>}>
    User Profile
  </Card.Header>
  <Card.Body>
    <ProfileForm user={user} />
  </Card.Body>
  <Card.Footer>
    <Button onClick={handleSave}>Save Changes</Button>
  </Card.Footer>
</Card>;
```

---

## 3. Context-Based Compound Components

When sub-components need implicit access to shared state from the parent:

```jsx
// The Context carries the shared state between parent and sub-components
const AccordionContext = createContext(null);

function Accordion({ children, defaultOpen = null }) {
  const [openItem, setOpenItem] = useState(defaultOpen);

  const toggle = useCallback((id) => {
    setOpenItem((current) => (current === id ? null : id));
  }, []);

  return (
    <AccordionContext.Provider value={{ openItem, toggle }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ id, children }) {
  return (
    <AccordionContext.Consumer>
      {({ openItem, toggle }) => (
        <div className={`accordion-item ${openItem === id ? "open" : ""}`}>
          {/* Pass id through another Context so nested Header/Panel can access it */}
          <AccordionItemContext.Provider
            value={{ id, isOpen: openItem === id, toggle }}
          >
            {children}
          </AccordionItemContext.Provider>
        </div>
      )}
    </AccordionContext.Consumer>
  );
}

// Sub-component reads from the nearest AccordionItemContext
function AccordionHeader({ children }) {
  const { id, isOpen, toggle } = useContext(AccordionItemContext);

  return (
    <button
      className="accordion-header"
      onClick={() => toggle(id)}
      aria-expanded={isOpen}
      aria-controls={`panel-${id}`}
    >
      {children}
      <ChevronIcon className={isOpen ? "rotate-180" : ""} />
    </button>
  );
}

function AccordionPanel({ children }) {
  const { id, isOpen } = useContext(AccordionItemContext);

  return (
    <div
      id={`panel-${id}`}
      role="region"
      hidden={!isOpen}
      className="accordion-panel"
    >
      {children}
    </div>
  );
}

// Attach as properties for discoverability
Accordion.Item = AccordionItem;
Accordion.Header = AccordionHeader;
Accordion.Panel = AccordionPanel;
```

---

## 4. Flexible Composition Without Rigid Structure

```jsx
// The caller controls structure; parent handles behavior
function FAQ({ items }) {
  return (
    <Accordion>
      {items.map((item) => (
        <Accordion.Item key={item.id} id={item.id}>
          {/* Structure is entirely up to the caller */}
          <div className="faq-item">
            <Accordion.Header>
              <span className="faq-category">{item.category}</span>
              {item.question}
            </Accordion.Header>
            <Accordion.Panel>
              <p>{item.answer}</p>
              {item.hasVideo && <VideoPlayer src={item.videoUrl} />}
              <a href={item.learnMoreUrl}>Learn more →</a>
            </Accordion.Panel>
          </div>
        </Accordion.Item>
      ))}
    </Accordion>
  );
}

// Compare to a non-compound API where you'd need:
<Accordion
  items={items.map((item) => ({
    id: item.id,
    header: (
      <>
        <span>{item.category}</span>
        {item.question}
      </>
    ),
    panel: (
      <>
        {item.answer}
        {item.hasVideo && <VideoPlayer />}
        <a href={item.url}>...</a>
      </>
    ),
  }))}
/>;
// Which is harder to read, harder to compose, and requires the accordion
// to accept ReactNode for both header AND panel — same end result but less clear
```

---

## 5. Controlled vs Uncontrolled Compound Components

Compound components should support both modes to be flexible:

```typescript
interface AccordionProps {
  // Controlled mode
  openItem?:  string | null;
  onChange?:  (id: string | null) => void;
  // Uncontrolled mode
  defaultOpenItem?: string | null;
  // Both
  children: ReactNode;
}

function Accordion({ openItem, onChange, defaultOpenItem, children }: AccordionProps) {
  const isControlled = openItem !== undefined;

  // Internal state for uncontrolled mode
  const [internalOpen, setInternalOpen] = useState<string | null>(defaultOpenItem ?? null);

  const effectiveOpen = isControlled ? openItem : internalOpen;

  const toggle = useCallback((id: string) => {
    const next = effectiveOpen === id ? null : id;
    if (!isControlled) setInternalOpen(next);
    onChange?.(next);
  }, [effectiveOpen, isControlled, onChange]);

  return (
    <AccordionContext.Provider value={{ openItem: effectiveOpen, toggle }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

// Usage as uncontrolled (internal state):
<Accordion defaultOpenItem="faq-1">
  <Accordion.Item id="faq-1">...</Accordion.Item>
  <Accordion.Item id="faq-2">...</Accordion.Item>
</Accordion>

// Usage as controlled (parent owns state):
const [openFAQ, setOpenFAQ] = useState(null);
<Accordion openItem={openFAQ} onChange={setOpenFAQ}>
  {/* Same children */}
</Accordion>
// Parent can now respond to accordion state (e.g., URL-based deep link)
```

---

## 6. Keyboard Navigation and ARIA

```jsx
const TabsContext = createContext(null);
const TabListContext = createContext(null);

function Tabs({ defaultValue, value, onChange, children }) {
  const isControlled = value !== undefined;
  const [internal, setInternal] = useState(defaultValue);
  const activeTab = isControlled ? value : internal;
  const tabListRef = useRef(null);
  const tabRefs = useRef(new Map()); // Map<id, HTMLButtonElement>

  const select = useCallback(
    (id) => {
      if (!isControlled) setInternal(id);
      onChange?.(id);
    },
    [isControlled, onChange],
  );

  // Keyboard navigation: arrow keys cycle through tabs
  const handleKeyDown = useCallback(
    (e) => {
      const tabs = [...tabRefs.current.entries()].filter(
        ([, el]) => !el.disabled,
      );
      const current = tabs.findIndex(([id]) => id === activeTab);

      let next = current;
      if (e.key === "ArrowRight") next = (current + 1) % tabs.length;
      if (e.key === "ArrowLeft")
        next = (current - 1 + tabs.length) % tabs.length;
      if (e.key === "Home") next = 0;
      if (e.key === "End") next = tabs.length - 1;

      if (next !== current) {
        e.preventDefault();
        const [nextId, nextEl] = tabs[next];
        select(nextId);
        nextEl.focus();
      }
    },
    [activeTab, select],
  );

  return (
    <TabsContext.Provider value={{ activeTab, select, tabRefs, handleKeyDown }}>
      {children}
    </TabsContext.Provider>
  );
}

function TabList({ children, label }) {
  const { handleKeyDown } = useContext(TabsContext);
  return (
    <div role="tablist" aria-label={label} onKeyDown={handleKeyDown}>
      {children}
    </div>
  );
}

function Tab({ id, children, disabled = false }) {
  const { activeTab, select, tabRefs } = useContext(TabsContext);
  const isActive = activeTab === id;

  return (
    <button
      ref={(el) => {
        if (el) tabRefs.current.set(id, el);
        else tabRefs.current.delete(id);
      }}
      role="tab"
      id={`tab-${id}`}
      aria-selected={isActive}
      aria-controls={`panel-${id}`}
      tabIndex={isActive ? 0 : -1} // roving tabIndex pattern
      disabled={disabled}
      onClick={() => !disabled && select(id)}
      className={`tab ${isActive ? "active" : ""}`}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }) {
  const { activeTab } = useContext(TabsContext);
  const isActive = activeTab === id;
  return (
    <div
      role="tabpanel"
      id={`panel-${id}`}
      aria-labelledby={`tab-${id}`}
      hidden={!isActive}
    >
      {children}
    </div>
  );
}

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;
```

---

## 7. Headless Components

Headless components separate behavior from presentation — all the logic, keyboard handling, and accessibility is provided; none of the visual design:

```jsx
// Headless: logic + accessibility, zero visual opinion
function HeadlessDisclosure({ defaultOpen = false, children }) {
  const [isOpen, setIsOpen] = useState(defaultOpen);
  const toggle = useCallback(() => setIsOpen(o => !o), []);
  const buttonId = useId();
  const panelId  = useId();

  return children({
    isOpen,
    toggle,
    // Spread these on the trigger element for correct ARIA
    getTriggerProps: () => ({
      id:               buttonId,
      type:             'button' as const,
      'aria-expanded':  isOpen,
      'aria-controls':  panelId,
      onClick:          toggle,
    }),
    // Spread these on the panel element for correct ARIA
    getPanelProps: () => ({
      id:                panelId,
      'aria-labelledby': buttonId,
      role:              'region' as const,
    }),
  });
}

// Usage: caller provides ALL the visuals
<HeadlessDisclosure>
  {({ isOpen, getTriggerProps, getPanelProps }) => (
    <div className="my-disclosure">
      <button {...getTriggerProps()} className="my-trigger">
        Show details
        {isOpen ? '▲' : '▼'}
      </button>
      {isOpen && (
        <div {...getPanelProps()} className="my-panel">
          <p>Content shown when open</p>
        </div>
      )}
    </div>
  )}
</HeadlessDisclosure>

// The same logic, completely different visual design:
<HeadlessDisclosure>
  {({ isOpen, getTriggerProps, getPanelProps }) => (
    <div style={{ border: '2px solid blue' }}>
      <h3 {...getTriggerProps()} style={{ cursor: 'pointer' }}>
        Click me
      </h3>
      {isOpen && (
        <p {...getPanelProps()}>
          Shown content
        </p>
      )}
    </div>
  )}
</HeadlessDisclosure>
```

---

## 8. Compound Components with TypeScript

```typescript
// Strongly typed compound component with context and sub-components

interface TabsContextValue {
  activeTab: string;
  select: (id: string) => void;
  tabRefs: React.MutableRefObject<Map<string, HTMLButtonElement>>;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(componentName: string): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (!ctx) {
    throw new Error(`${componentName} must be used within a Tabs component`);
  }
  return ctx;
}

// Root component with type-safe props
interface TabsProps {
  defaultValue?: string;
  value?: string;
  onChange?: (value: string) => void;
  children: ReactNode;
}

function Tabs(props: TabsProps) {
  /* ... implementation */
}

// Sub-component interfaces
interface TabProps {
  id: string;
  disabled?: boolean;
  children: ReactNode;
  className?: string;
}

function Tab({ id, disabled = false, children, className }: TabProps) {
  const { activeTab, select } = useTabsContext("Tab");
  // ...
}

// Attach sub-components with correct types
interface TabsComposition {
  List: typeof TabList;
  Tab: typeof Tab;
  Panel: typeof TabPanel;
}

const Tabs = Object.assign(TabsRoot, {
  List: TabList,
  Tab: Tab,
  Panel: TabPanel,
} satisfies TabsComposition);

export { Tabs };
```

---

## 9. Real-World Example — Accordion

```tsx
// Production-ready Accordion compound component
const AccordionCtx = createContext<{
  openItems: Set<string>;
  toggle: (id: string) => void;
  type: "single" | "multiple";
} | null>(null);

const AccordionItemCtx = createContext<{
  id: string;
  isOpen: boolean;
} | null>(null);

interface AccordionProps {
  type?: "single" | "multiple";
  defaultValue?: string | string[];
  value?: string | string[];
  onChange?: (value: string | string[]) => void;
  children: ReactNode;
  className?: string;
}

function Accordion({
  type = "single",
  defaultValue,
  value,
  onChange,
  children,
  className,
}: AccordionProps) {
  const isControlled = value !== undefined;

  const [internalOpen, setInternalOpen] = useState<Set<string>>(() => {
    const initial = defaultValue
      ? Array.isArray(defaultValue)
        ? defaultValue
        : [defaultValue]
      : [];
    return new Set(initial);
  });

  const openItems = isControlled
    ? new Set(Array.isArray(value) ? value : value ? [value] : [])
    : internalOpen;

  const toggle = useCallback(
    (id: string) => {
      let next: Set<string>;

      if (type === "single") {
        next = openItems.has(id) ? new Set() : new Set([id]);
      } else {
        next = new Set(openItems);
        if (next.has(id)) next.delete(id);
        else next.add(id);
      }

      if (!isControlled) setInternalOpen(next);

      if (onChange) {
        onChange(type === "single" ? ([...next][0] ?? "") : [...next]);
      }
    },
    [openItems, type, isControlled, onChange],
  );

  return (
    <AccordionCtx.Provider value={{ openItems, toggle, type }}>
      <div className={className}>{children}</div>
    </AccordionCtx.Provider>
  );
}

function AccordionItem({
  id,
  children,
  className,
}: {
  id: string;
  children: ReactNode;
  className?: string;
}) {
  const ctx = useContext(AccordionCtx)!;
  const isOpen = ctx.openItems.has(id);

  return (
    <AccordionItemCtx.Provider value={{ id, isOpen }}>
      <div className={className} data-state={isOpen ? "open" : "closed"}>
        {children}
      </div>
    </AccordionItemCtx.Provider>
  );
}

function AccordionTrigger({
  children,
  className,
}: {
  children: ReactNode;
  className?: string;
}) {
  const { id, isOpen } = useContext(AccordionItemCtx)!;
  const { toggle } = useContext(AccordionCtx)!;

  return (
    <button
      type="button"
      aria-expanded={isOpen}
      aria-controls={`accordion-content-${id}`}
      onClick={() => toggle(id)}
      className={className}
      data-state={isOpen ? "open" : "closed"}
    >
      {children}
    </button>
  );
}

function AccordionContent({
  children,
  className,
}: {
  children: ReactNode;
  className?: string;
}) {
  const { id, isOpen } = useContext(AccordionItemCtx)!;

  return (
    <div
      id={`accordion-content-${id}`}
      role="region"
      aria-labelledby={`accordion-trigger-${id}`}
      hidden={!isOpen}
      className={className}
      data-state={isOpen ? "open" : "closed"}
    >
      {children}
    </div>
  );
}

// Attach as sub-components
Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

// Usage: clean, flexible, accessible
<Accordion type="single" defaultValue="item-1">
  {faqs.map((faq) => (
    <Accordion.Item key={faq.id} id={faq.id} className="faq-item">
      <Accordion.Trigger className="faq-trigger">
        {faq.question}
        <ChevronIcon />
      </Accordion.Trigger>
      <Accordion.Content className="faq-content">
        <p>{faq.answer}</p>
      </Accordion.Content>
    </Accordion.Item>
  ))}
</Accordion>;
```

---

## 10. Real-World Example — Tabs

```tsx
// Usage of the full Tabs compound component from Section 6:
<Tabs defaultValue="overview">
  <Tabs.List label="Product details">
    <Tabs.Tab id="overview">Overview</Tabs.Tab>
    <Tabs.Tab id="specs">Specifications</Tabs.Tab>
    <Tabs.Tab id="reviews">
      Reviews
      <Badge>{reviewCount}</Badge>
    </Tabs.Tab>
    <Tabs.Tab id="shipping">Shipping & Returns</Tabs.Tab>
  </Tabs.List>

  <Tabs.Panel id="overview">
    <ProductOverview product={product} />
  </Tabs.Panel>
  <Tabs.Panel id="specs">
    <SpecsTable specs={product.specs} />
  </Tabs.Panel>
  <Tabs.Panel id="reviews">
    <ReviewList productId={product.id} />
  </Tabs.Panel>
  <Tabs.Panel id="shipping">
    <ShippingInfo shippingZone={product.shippingZone} />
  </Tabs.Panel>
</Tabs>
```

---

## 11. Good Practices

### ✅ Throw a clear error when sub-components are used outside parent

```typescript
// ✅ Actionable error message
function useTabsCtx() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error("<Tab> must be used inside a <Tabs> component");
  return ctx;
}
```

### ✅ Support both controlled and uncontrolled modes

```typescript
// ✅ All compound components should support both modes for flexibility
// Controlled: value + onChange
// Uncontrolled: defaultValue (or defaultOpen, defaultSelected, etc.)
```

### ✅ Use data attributes for CSS hooks

```jsx
// ✅ Consumers can style based on state without reading JS
<AccordionItem data-state={isOpen ? 'open' : 'closed'}>
// CSS:
// [data-state='open'] .accordion-icon { transform: rotate(180deg); }
```

---

## 12. Bad Practices

### ❌ Brittle children.map() type-checking

```jsx
// ❌ Checking child.type is fragile (breaks with memoization, forwardRef wrappers)
function Tabs({ children }) {
  React.Children.map(children, (child) => {
    if (child.type !== Tab) throw new Error("Only Tab children allowed"); // fragile!
  });
}

// ✅ Use Context implicitly — any component that calls useTabsContext() works
// The parent doesn't need to inspect its children's types
```

### ❌ Deep Context nesting without useMemo

```jsx
// ❌ New context value object on every render → all consumers re-render unnecessarily
function Accordion({ children }) {
  const [openItem, setOpenItem] = useState(null);
  return (
    <AccordionContext.Provider
      value={{
        openItem,
        toggle: (id) => setOpenItem((i) => (i === id ? null : id)),
      }} // new obj every render!
    >
      {children}
    </AccordionContext.Provider>
  );
}

// ✅ Memoize the context value
function Accordion({ children }) {
  const [openItem, setOpenItem] = useState(null);
  const toggle = useCallback(
    (id) => setOpenItem((i) => (i === id ? null : id)),
    [],
  );
  const value = useMemo(() => ({ openItem, toggle }), [openItem, toggle]);

  return (
    <AccordionContext.Provider value={value}>
      {children}
    </AccordionContext.Provider>
  );
}
```

---

## 13. Common Mistakes

### Mistake 1 — Forgetting to handle missing context gracefully

```jsx
// ❌ Cryptic error if Tab is used outside Tabs
function Tab({ id }) {
  const { activeTab } = useContext(TabsContext); // null if outside — crashes with "can't destructure null"
}

// ✅ Clear error message
function Tab({ id }) {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error("<Tab> must be rendered inside <Tabs>");
  const { activeTab } = ctx;
}
```

### Mistake 2 — Sub-component state not reset on parent remount

```jsx
// ❌ Stale state if parent component persists while data changes
<Accordion defaultOpen="item-1">
  {dynamicItems.map(item => <Accordion.Item id={item.id} />)}
</Accordion>
// If dynamicItems changes: defaultOpen was already consumed on first mount,
// the accordion doesn't reset to reflect the new first item

// ✅ Key the parent to force remount on data changes
<Accordion key={dynamicItems[0]?.id} defaultOpen={dynamicItems[0]?.id}>
  {dynamicItems.map(item => <Accordion.Item id={item.id} />)}
</Accordion>
```

---

## 14. Interview-Level Explanation

> **"What is the compound component pattern? When would you use it?"**

**Strong answer:**

> "The compound component pattern builds APIs where a parent component manages shared state and behavior, and a set of child components access that state implicitly via React Context — without any of them needing to receive the state as explicit props. The caller assembles the sub-components freely: they can reorder them, omit some, or put custom elements in between, while the shared behavior — toggle state, keyboard navigation, ARIA relationships — remains correct throughout.
>
> The canonical example is a Tabs component. The parent tracks which tab is active. The `Tab` sub-components need to know whether they're active (for styling and tabIndex), and the `TabPanel` sub-components need to know whether to show or hide. Both get this through a shared Context that the parent provides. The caller doesn't have to manage any state — they just compose the pieces, and everything works.
>
> The pattern excels at what I'd call 'semantic structure with flexible presentation.' The component handles the hard parts — keyboard navigation according to ARIA authoring practices, correct aria-selected/aria-controls/aria-labelledby relationships, focus management — while the caller has complete freedom over visual design and DOM structure. This is exactly the model Radix UI, Headless UI, and Reach UI use: they provide behavior and accessibility; you provide the design.
>
> The implementation typically involves a Context at the root that holds the shared state and action callbacks, an Item-level Context for sub-state that's scoped to each item (like which accordion item is currently in scope), and a custom hook like `useTabsContext()` that reads the Context and throws a helpful error if called outside the parent.
>
> The key design decisions are: supporting both controlled and uncontrolled modes (so the parent can either manage its own state or be driven by the caller), using `data-state` attributes for styling hooks (so consumers can target open/closed states in CSS without reading JS state), and memoizing the context value to avoid unnecessary re-renders of all consumers on every render of the root component."

---

## 15. Exercises

### Exercise 1 — Build a compound Toggle component

Build a compound `Toggle` component with:

- `Toggle.Trigger`: a button that toggles the state
- `Toggle.On`: renders its children only when toggled on
- `Toggle.Off`: renders its children only when toggled off
- Controlled and uncontrolled modes
- `aria-pressed` on the trigger

<details>
<summary>Solution</summary>

```tsx
interface ToggleContextValue {
  isOn: boolean;
  toggle: () => void;
}

const ToggleContext = createContext<ToggleContextValue | null>(null);

function useToggleCtx() {
  const ctx = useContext(ToggleContext);
  if (!ctx)
    throw new Error("Toggle sub-components must be used within <Toggle>");
  return ctx;
}

// Root component
interface ToggleProps {
  on?: boolean; // controlled
  defaultOn?: boolean; // uncontrolled
  onChange?: (on: boolean) => void;
  children: ReactNode;
}

function Toggle({ on, defaultOn = false, onChange, children }: ToggleProps) {
  const isControlled = on !== undefined;
  const [internal, setInternal] = useState(defaultOn);
  const isOn = isControlled ? on! : internal;

  const toggle = useCallback(() => {
    const next = !isOn;
    if (!isControlled) setInternal(next);
    onChange?.(next);
  }, [isOn, isControlled, onChange]);

  const value = useMemo(() => ({ isOn, toggle }), [isOn, toggle]);

  return (
    <ToggleContext.Provider value={value}>{children}</ToggleContext.Provider>
  );
}

// Sub-components
function ToggleTrigger({
  children,
  className,
}: {
  children: ReactNode;
  className?: string;
}) {
  const { isOn, toggle } = useToggleCtx();
  return (
    <button
      type="button"
      aria-pressed={isOn}
      onClick={toggle}
      className={className}
      data-state={isOn ? "on" : "off"}
    >
      {children}
    </button>
  );
}

function ToggleOn({ children }: { children: ReactNode }) {
  const { isOn } = useToggleCtx();
  return isOn ? <>{children}</> : null;
}

function ToggleOff({ children }: { children: ReactNode }) {
  const { isOn } = useToggleCtx();
  return isOn ? null : <>{children}</>;
}

// Attach sub-components
Toggle.Trigger = ToggleTrigger;
Toggle.On = ToggleOn;
Toggle.Off = ToggleOff;

// Usage:
<Toggle defaultOn={false} onChange={(on) => console.log("toggled:", on)}>
  <Toggle.Trigger className="theme-button">
    <Toggle.On>🌙 Dark Mode</Toggle.On>
    <Toggle.Off>☀️ Light Mode</Toggle.Off>
  </Toggle.Trigger>

  <Toggle.On>
    <DarkModeStyles />
  </Toggle.On>
  <Toggle.Off>
    <LightModeStyles />
  </Toggle.Off>
</Toggle>;
```

</details>

---

## 🔗 Related Topics

- [`patterns/01-component-composition.md`](./01-component-composition.md) — Foundation of composition patterns
- [`patterns/02-custom-hooks.md`](./02-custom-hooks.md) — useContext + custom hooks underpinning compound components
- [`patterns/03-render-props-hoc.md`](./03-render-props-hoc.md) — Pre-compound-component patterns
- [`patterns/04-controlled-uncontrolled.md`](./04-controlled-uncontrolled.md) — Controlled/uncontrolled in component design

---

<div align="center">

**`patterns/` section complete!** 🎉

All 5 patterns files done:
`01-component-composition.md` · `02-custom-hooks.md` · `03-render-props-hoc.md` · `04-controlled-uncontrolled.md` · **`05-compound-components.md`** ✓

**Next section:** [`anti-patterns/`](../anti-patterns/) →

</div>
