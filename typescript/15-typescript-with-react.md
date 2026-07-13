# 15 — TypeScript with React

> **"TypeScript and React are designed for each other. Props become contracts. Hooks have precise return types. Events know their target. But the patterns that make this combination work well — generic components, properly typed forwardRef, ComponentProps inference — are not obvious from the documentation alone. This file is the practical guide to getting them right."**

🟠 **Level: Advanced**

---

## 📚 Table of Contents

1. [Typing Props](#1-typing-props)
2. [Children Patterns](#2-children-patterns)
3. [Typing Hooks](#3-typing-hooks)
4. [Event Types](#4-event-types)
5. [Generic Components](#5-generic-components)
6. [forwardRef with TypeScript](#6-forwardref-with-typescript)
7. [ComponentProps and HTML Attribute Extension](#7-componentprops-and-html-attribute-extension)
8. [Context with TypeScript](#8-context-with-typescript)
9. [Discriminated Union Props](#9-discriminated-union-props)
10. [Common Mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)

---

## 1. Typing Props

```typescript
// Basic props: use type or interface (both work, teams pick one convention)
type ButtonProps = {
  label:     string;
  onClick:   () => void;
  disabled?: boolean;
  variant?:  'primary' | 'secondary' | 'danger';
  size?:     'sm' | 'md' | 'lg';
};

function Button({ label, onClick, disabled = false, variant = 'primary', size = 'md' }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant} btn-${size}`}
    >
      {label}
    </button>
  );
}

// React.FC (function component) type — optional, controversial
// Pros: includes displayName, defaultProps type handling
// Cons: historically had unwanted implicit children, reduced inference
// Recommendation: skip React.FC, just annotate parameters directly (as above)

// With return type annotation (usually optional — inferred)
function Greeting({ name }: { name: string }): JSX.Element {
  return <h1>Hello, {name}!</h1>;
}

// React.ReactElement vs JSX.Element vs React.ReactNode:
// JSX.Element      = result of a JSX expression (React.ReactElement<any, any>)
// React.ReactElement = a rendered React element
// React.ReactNode  = anything React can render (element, string, number, null, array, etc.)
```

---

## 2. Children Patterns

```typescript
import { ReactNode, PropsWithChildren, ReactElement } from 'react';

// 1. ReactNode: most permissive — use for general children
type ContainerProps = {
  title:    string;
  children: ReactNode; // string | ReactElement | number | null | boolean | ...
};

// 2. PropsWithChildren utility type
type CardProps = PropsWithChildren<{
  title: string;
  footer?: ReactNode;
}>;
// equivalent to: { children?: ReactNode; title: string; footer?: ReactNode }

// 3. Specific child type (compound components)
type TabsProps = {
  children: ReactElement<TabProps> | ReactElement<TabProps>[];
};

// 4. Render prop pattern
type DataFetcherProps<T> = {
  url:      string;
  render:   (data: T, isLoading: boolean) => ReactNode;
};

function DataFetcher<T>({ url, render }: DataFetcherProps<T>) {
  const { data, isLoading } = useFetch<T>(url);
  return <>{render(data, isLoading)}</>;
}

// 5. Children as a function (function-as-children pattern)
type AnimatedProps = {
  children: (progress: number) => ReactNode;
  duration: number;
};

// Usage:
<Animated duration={300}>
  {(progress) => <div style={{ opacity: progress }}>Fading in</div>}
</Animated>
```

---

## 3. Typing Hooks

```typescript
// useState: usually inferred, but sometimes needs annotation
const [count, setCount] = useState(0); // number inferred
const [user, setUser] = useState<User | null>(null); // explicit when initial is null
const [items, setItems] = useState<string[]>([]); // explicit for empty array

// useRef: three different signatures
const inputRef = useRef<HTMLInputElement>(null); // DOM element — starts as null
const valueRef = useRef<number>(0); // mutable value — non-null
const counterRef = useRef<{ value: number }>({ value: 0 }); // object ref

inputRef.current?.focus(); // current is HTMLInputElement | null

// useReducer
type State = { count: number; status: "idle" | "loading" | "done" };
type Action =
  | { type: "INCREMENT" }
  | { type: "SET_STATUS"; status: State["status"] };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1 };
    case "SET_STATUS":
      return { ...state, status: action.status };
  }
}
const [state, dispatch] = useReducer(reducer, { count: 0, status: "idle" });

// Custom hook return types
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle]; // inferred as (boolean | (() => void))[] — needs annotation!
}
// OR use `as const`:
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle] as const; // [boolean, () => void] ✅
}

// Custom hook with object return (usually clearer)
function useAsync<T>(fn: () => Promise<T>) {
  const [state, setState] = useState<{
    data: T | null;
    error: Error | null;
    status: "idle" | "loading" | "success" | "error";
  }>({ data: null, error: null, status: "idle" });

  const execute = useCallback(async () => {
    setState((s) => ({ ...s, status: "loading" }));
    try {
      const data = await fn();
      setState({ data, error: null, status: "success" });
    } catch (error) {
      setState({ data: null, error: error as Error, status: "error" });
    }
  }, [fn]);

  return { ...state, execute };
}
```

---

## 4. Event Types

```typescript
// React event types: React.[Event]<[Element]>
// The most important ones:

import {
  MouseEvent, ChangeEvent, FormEvent, KeyboardEvent,
  FocusEvent, DragEvent, PointerEvent, TouchEvent
} from 'react';

// Click handler
function handleClick(e: MouseEvent<HTMLButtonElement>): void {
  e.currentTarget; // HTMLButtonElement ✅
  e.clientX;       // number ✅
}

// Input change
function handleChange(e: ChangeEvent<HTMLInputElement>): void {
  e.target.value;  // string ✅
}

// Textarea change
function handleTextarea(e: ChangeEvent<HTMLTextAreaElement>): void {
  e.target.value;  // string ✅
}

// Select change
function handleSelect(e: ChangeEvent<HTMLSelectElement>): void {
  e.target.value;    // string ✅
  e.target.options;  // HTMLOptionsCollection ✅
}

// Form submit
function handleSubmit(e: FormEvent<HTMLFormElement>): void {
  e.preventDefault(); // ✅
  const form = e.currentTarget;
  const data = new FormData(form); // ✅
}

// Keyboard
function handleKeyDown(e: KeyboardEvent<HTMLInputElement>): void {
  e.key;     // string ✅
  e.code;    // string ✅
  e.ctrlKey; // boolean ✅
}

// Shorthand: inline event handlers (usually inferred)
<input
  onChange={(e) => console.log(e.target.value)} // e is ChangeEvent<HTMLInputElement>
  onKeyDown={(e) => { if (e.key === 'Enter') submit(); }}
  onFocus={(e) => e.target.select()}
/>

// When storing handlers in state or passing as prop
type ClickHandler = MouseEvent<HTMLButtonElement>;
type ChangeHandler = ChangeEvent<HTMLInputElement>;
```

---

## 5. Generic Components

```typescript
// A generic component that works with any data type

// Generic list component
type ListProps<T> = {
  items:      T[];
  renderItem: (item: T, index: number) => ReactNode;
  keyExtractor: (item: T) => string | number;
  emptyState?: ReactNode;
};

function List<T>({ items, renderItem, keyExtractor, emptyState }: ListProps<T>) {
  if (items.length === 0) return <>{emptyState ?? <p>No items</p>}</>;
  return (
    <ul>
      {items.map((item, i) => (
        <li key={keyExtractor(item)}>{renderItem(item, i)}</li>
      ))}
    </ul>
  );
}

// Usage — T is inferred from `items`:
<List
  items={users}               // T inferred as User
  keyExtractor={u => u.id}    // u: User ✅
  renderItem={u => u.name}    // u: User ✅
/>

// Generic select component
type SelectProps<T> = {
  options:     T[];
  value:       T | null;
  onChange:    (value: T) => void;
  getLabel:    (option: T) => string;
  getValue:    (option: T) => string | number;
};

function Select<T>({ options, value, onChange, getLabel, getValue }: SelectProps<T>) {
  return (
    <select
      value={value ? String(getValue(value)) : ''}
      onChange={(e) => {
        const selected = options.find(o => String(getValue(o)) === e.target.value);
        if (selected) onChange(selected);
      }}
    >
      {options.map(option => (
        <option key={getValue(option)} value={getValue(option)}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  );
}

// Generic data table
type Column<T> = {
  key:       keyof T | string;
  header:    string;
  render?:   (row: T) => ReactNode;
  accessor?: (row: T) => ReactNode;
};

type TableProps<T extends { id: string | number }> = {
  columns: Column<T>[];
  data:    T[];
};
```

---

## 6. forwardRef with TypeScript

```typescript
import { forwardRef, useImperativeHandle, ForwardedRef, Ref } from 'react';

// Basic forwardRef
const Input = forwardRef<HTMLInputElement, InputProps>(
  function Input({ label, ...props }, ref) {
    return (
      <div>
        <label>{label}</label>
        <input ref={ref} {...props} />
      </div>
    );
  }
);

// With useImperativeHandle (exposing a custom API)
interface InputHandle {
  focus():  void;
  clear():  void;
  getValue(): string;
}

type ManagedInputProps = {
  label:        string;
  defaultValue?: string;
};

const ManagedInput = forwardRef<InputHandle, ManagedInputProps>(
  function ManagedInput({ label, defaultValue = '' }, ref) {
    const inputRef = useRef<HTMLInputElement>(null);

    useImperativeHandle(ref, () => ({
      focus() { inputRef.current?.focus(); },
      clear() { if (inputRef.current) inputRef.current.value = ''; },
      getValue() { return inputRef.current?.value ?? ''; },
    }));

    return (
      <div>
        <label>{label}</label>
        <input ref={inputRef} defaultValue={defaultValue} />
      </div>
    );
  }
);

// Using it:
const ref = useRef<InputHandle>(null);
ref.current?.focus();   // ✅
ref.current?.getValue(); // ✅ string

// Generic forwardRef (TypeScript limitation workaround)
// forwardRef doesn't forward generic type parameters out of the box
// Workaround: cast after creation
function genericForwardRef<T, P = {}>(
  render: (props: P, ref: Ref<T>) => ReactElement | null
): (props: P & { ref?: Ref<T> }) => ReactElement | null {
  return forwardRef(render) as any;
}

const GenericList = genericForwardRef(
  function <T>(
    { items }: { items: T[] },
    ref: ForwardedRef<HTMLUListElement>
  ) {
    return <ul ref={ref}>{/* ... */}</ul>;
  }
);
```

---

## 7. ComponentProps and HTML Attribute Extension

```typescript
import { ComponentProps, ComponentPropsWithRef, ButtonHTMLAttributes } from 'react';

// Extend native HTML element props (most common pattern for UI library components)
type ButtonProps = ButtonHTMLAttributes<HTMLButtonElement> & {
  variant?: 'primary' | 'secondary' | 'danger';
  isLoading?: boolean;
};

function Button({ variant = 'primary', isLoading, children, ...htmlProps }: ButtonProps) {
  return (
    <button
      {...htmlProps}           // forward ALL native button props (className, disabled, etc.)
      disabled={isLoading || htmlProps.disabled}
      className={`btn btn-${variant} ${htmlProps.className ?? ''}`}
    >
      {isLoading ? <Spinner /> : children}
    </button>
  );
}
// Consumer gets autocomplete for ALL button HTML attributes + your custom ones ✅

// Using ComponentProps to derive types from existing components
type InputProps = ComponentProps<'input'>;         // all native input props
type DivProps   = ComponentProps<'div'>;           // all native div props
type MyButtonExtended = ComponentProps<typeof Button>; // all Button props

// ComponentPropsWithRef: includes the ref prop
type InputWithRef = ComponentPropsWithRef<'input'>; // includes ref: Ref<HTMLInputElement>

// Polymorphic component (the `as` prop pattern)
type AsProps<E extends ElementType = ElementType> = {
  as?: E;
};
type PolymorphicProps<E extends ElementType, P = {}> =
  AsProps<E> &
  Omit<ComponentPropsWithRef<E>, keyof P> &
  P;

function Text<E extends ElementType = 'span'>({
  as,
  ...props
}: PolymorphicProps<E, { className?: string }>) {
  const Component = as ?? 'span';
  return <Component {...props} />;
}

<Text as="h1">Heading</Text>       // renders as h1, TypeScript knows h1 props ✅
<Text as="label" htmlFor="email">  // htmlFor valid on label ✅
  Email
</Text>
```

---

## 8. Context with TypeScript

```typescript
import { createContext, useContext, ReactNode } from 'react';

// Pattern 1: Non-null context (throw if used outside provider)
interface AuthContextValue {
  user:   User | null;
  login:  (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}

function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (credentials: Credentials) => {
    const user = await authApi.login(credentials);
    setUser(user);
  };
  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Pattern 2: Context with default value (no null check needed)
const ThemeContext = createContext<'light' | 'dark'>('light');
const useTheme = () => useContext(ThemeContext); // 'light' | 'dark', never null

// Pattern 3: Generic context factory
function createCtx<T>() {
  const ctx = createContext<T | undefined>(undefined);
  function useCtx(): T {
    const c = useContext(ctx);
    if (!c) throw new Error('useCtx must be inside Provider');
    return c;
  }
  return [ctx, useCtx] as const;
}

const [UserCtx, useUser] = createCtx<User>();
```

---

## 9. Discriminated Union Props

```typescript
// Force mutually exclusive prop combinations using discriminated unions

// ❌ Problem: both onClick and href can be passed together
type BadButtonProps = {
  label:   string;
  href?:   string;
  onClick?: () => void;
};

// ✅ Solution: discriminated union — only one of href/onClick is valid
type ButtonLinkProps =
  | { label: string; href: string; onClick?: never }   // link mode
  | { label: string; onClick: () => void; href?: never }; // button mode

function ButtonOrLink(props: ButtonLinkProps) {
  if (props.href) {
    return <a href={props.href}>{props.label}</a>;
  }
  return <button onClick={props.onClick}>{props.label}</button>;
}

<ButtonOrLink label="Home" href="/" />           // ✅ link mode
<ButtonOrLink label="Submit" onClick={submit} />  // ✅ button mode
// <ButtonOrLink label="Bad" href="/" onClick={fn} />  // ❌ both provided

// Another example: controlled vs uncontrolled input
type InputProps =
  | {
      controlled: true;
      value:       string;
      onChange:    (value: string) => void;
      defaultValue?: never;
    }
  | {
      controlled?: false;
      defaultValue?: string;
      value?:      never;
      onChange?:   never;
    };

function Input({ controlled, ...props }: InputProps & { placeholder?: string }) {
  if (controlled) {
    return <input value={props.value} onChange={e => props.onChange(e.target.value)} />;
  }
  return <input defaultValue={props.defaultValue} />;
}
```

---

## 10. Common Mistakes

### Mistake 1 — Typing event handlers wrong

```typescript
// ❌ Too broad — loses type safety
function handleChange(e: Event) {
  (e.target as HTMLInputElement).value; // cast needed
}

// ✅ Use React's typed event
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  e.target.value; // string, no cast needed ✅
}
```

### Mistake 2 — Using React.FC loses generic component types

```typescript
// ❌ React.FC can't be made generic without workarounds
const List: React.FC<ListProps<any>> = ({ items }) => {
  /* ... */
};
// Generic T is lost — items is always `any[]`

// ✅ Skip React.FC, use a regular generic function
function List<T>({ items }: ListProps<T>) {
  /* ... */
}
```

### Mistake 3 — Refs not initialized correctly

```typescript
// ❌ Non-null ref for DOM element (DOM never puts values here — React does)
const ref = useRef<HTMLDivElement>({}); // wrong — DOM refs start null

// ✅ DOM refs should start null
const ref = useRef<HTMLDivElement>(null);

// ✅ Mutable value refs can start with a value
const valueRef = useRef<number>(0); // NOT a DOM ref — starts with value
```

---

## 11. Exercises

### Exercise 1 — Type a generic Table component

```typescript
// Build a Table<T> component with:
// - columns: Array<{ key: keyof T; header: string; render?: (row: T) => ReactNode }>
// - data: T[]
// - onRowClick?: (row: T) => void
// T must have an 'id' field (string | number) for keys
```

<details>
<summary>Solution</summary>

```typescript
import { ReactNode } from 'react';

type Column<T> = {
  key:     keyof T;
  header:  string;
  render?: (row: T) => ReactNode;
};

type TableProps<T extends { id: string | number }> = {
  columns:     Column<T>[];
  data:        T[];
  onRowClick?: (row: T) => void;
};

function Table<T extends { id: string | number }>({
  columns, data, onRowClick,
}: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={row.id} onClick={() => onRowClick?.(row)}>
            {columns.map(col => (
              <td key={String(col.key)}>
                {col.render ? col.render(row) : String(row[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage:
<Table
  data={users}
  columns={[
    { key: 'name', header: 'Name' },
    { key: 'email', header: 'Email' },
    { key: 'role', header: 'Role', render: u => <Badge>{u.role}</Badge> },
  ]}
  onRowClick={(user) => navigate(`/users/${user.id}`)}
/>
```

</details>

---

## 🔗 Related Topics

- [`07-generics.md`](./07-generics.md) — Generic patterns used in components
- [`08-utility-types.md`](./08-utility-types.md) — ComponentProps, ReturnType
- [`patterns/`](../patterns/) — React patterns with TypeScript examples
