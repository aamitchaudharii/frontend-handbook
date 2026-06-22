# 04 — Controlled and Uncontrolled Components

> **"Controlled components let you reason about your form state in React terms — you always know what's in every field because you put it there. Uncontrolled components let the DOM own the state — less code, but less control. The pattern you choose shapes your entire form architecture. Getting it right the first time is worth the thought."**

Controlled and uncontrolled are two fundamental patterns for managing form and input state in React. In a controlled component, React state is the single source of truth — every keystroke flows through `onChange` → `setState`. In an uncontrolled component, the DOM is the source of truth — React reads values from the DOM via refs only when needed. Each has a distinct character: controlled components are verbose but predictable; uncontrolled components are terse but harder to synchronize. This document covers both patterns, the tradeoffs, the hybrid patterns that real-world form libraries use, and the decision framework for choosing between them.

---

## 📚 Table of Contents

1. [Controlled Components](#1-controlled-components)
2. [Uncontrolled Components](#2-uncontrolled-components)
3. [The Key Differences](#3-the-key-differences)
4. [When to Use Each](#4-when-to-use-each)
5. [Partially Controlled (Hybrid) Patterns](#5-partially-controlled-hybrid-patterns)
6. [Defaulting and Re-Initializing](#6-defaulting-and-re-initializing)
7. [Building a Controlled Form Library Interface](#7-building-a-controlled-form-library-interface)
8. [React Hook Form — Uncontrolled with Schema](#8-react-hook-form--uncontrolled-with-schema)
9. [File Inputs and Other Special Cases](#9-file-inputs-and-other-special-cases)
10. [Controlled Components in Custom UI](#10-controlled-components-in-custom-ui)
11. [Good Practices](#11-good-practices)
12. [Bad Practices](#12-bad-practices)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview-Level Explanation](#14-interview-level-explanation)
15. [Exercises](#15-exercises)

---

## 1. Controlled Components

In a controlled component, React state drives the input value — the component controls the DOM.

```jsx
function ControlledForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  function handleSubmit(e) {
    e.preventDefault();
    // Values are immediately available from state — no DOM querying needed
    submitForm({ name, email });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={name} // React controls the DOM value
        onChange={(e) => setName(e.target.value)} // every keystroke updates state
      />
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Controlled Input Lifecycle

```
USER TYPES → browser fires onChange event → React handler calls setState
→ React re-renders → React updates the DOM input value → user sees their input

CYCLE TIME: ~1ms (imperceptible)
WHAT REACT OWNS: the source of truth is React state
WHAT DOM OWNS: a reflective copy of the React state value
```

### Controlled Pattern Capabilities

```jsx
// Because React owns the value, you can:

// 1. Validate on every keystroke
const [email, setEmail] = useState('');
const isValidEmail = email.includes('@');

<input
  value={email}
  onChange={e => setEmail(e.target.value)}
  style={{ borderColor: email && !isValidEmail ? 'red' : 'initial' }}
/>

// 2. Transform input before storing
const [price, setPrice] = useState('');
<input
  type="text"
  value={price}
  onChange={e => {
    const num = e.target.value.replace(/[^0-9.]/g, ''); // allow only numbers
    setPrice(num);
  }}
/>

// 3. Programmatically set values
<button onClick={() => setEmail('test@example.com')}>Fill test email</button>
// The input immediately shows the new value

// 4. Derive one value from another
const [username, setUsername] = useState('');
const [email, setEmail] = useState('');
// Auto-fill email from username:
<input
  value={username}
  onChange={e => {
    setUsername(e.target.value);
    if (!email) setEmail(`${e.target.value}@example.com`);
  }}
/>
```

---

## 2. Uncontrolled Components

In an uncontrolled component, the DOM is the source of truth. React reads values only when needed, via refs.

```jsx
function UncontrolledForm() {
  const nameRef = useRef(null);
  const emailRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    // Read values from DOM at submit time only
    const name = nameRef.current.value;
    const email = emailRef.current.value;
    submitForm({ name, email });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        ref={nameRef}
        defaultValue="" // sets initial value — NOT controlled by React after mount
      />
      <input type="email" ref={emailRef} defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Uncontrolled Input Lifecycle

```
USER TYPES → DOM updates directly → React is NOT notified (no re-render)
AT SUBMIT: React reads DOM value via ref → submits form data

NO RE-RENDERS during typing
WHAT REACT OWNS: nothing — DOM manages the value independently
WHAT DOM OWNS: the actual current value
```

### When Uncontrolled Works Well

```jsx
// Simple forms where you only need values at submit time
function SimpleLoginForm({ onLogin }) {
  const formRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(formRef.current));
    // { email: 'user@example.com', password: 'secret' }
    onLogin(data);
  }

  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button type="submit">Login</button>
    </form>
  );
}
// FormData API reads all named inputs without refs per-field
```

---

## 3. The Key Differences

```
                    CONTROLLED              UNCONTROLLED
Source of truth:    React state             DOM
Re-renders on type: Yes (every keystroke)   No
Access to value:    Always (via state)      At read time (via ref/FormData)
Set value prog.:    Yes (setState)          Limited (ref.current.value, but awkward)
Transform input:    Yes (in onChange)       No (must use DOM APIs)
Instant validation: Yes (in onChange)       No (only on blur/submit)
Reset to default:   Easy (setState(''))     Hard (must use ref or reset())
Performance:        1 re-render per key     No re-renders during typing
Form library use:   React state libs        React Hook Form (read at submit)
```

---

## 4. When to Use Each

```
USE CONTROLLED WHEN:
  ✓ Real-time validation as the user types
  ✓ Dependent fields (field B depends on field A's value)
  ✓ Programmatic control over values (fill from clipboard, auto-complete)
  ✓ Formatting as you type (credit card, phone number, date)
  ✓ Dynamic enable/disable based on input values
  ✓ The form state needs to exist outside the form (e.g., in a wizard)

USE UNCONTROLLED WHEN:
  ✓ Simple, one-time data collection (login forms, simple signup)
  ✓ Performance matters and the form is large (hundreds of fields)
  ✓ Integrating with non-React code that owns the form DOM
  ✓ File inputs (always uncontrolled — React can't set a file input value)
  ✓ Using React Hook Form or a similar uncontrolled-first library
```

---

## 5. Partially Controlled (Hybrid) Patterns

### Semi-Controlled: Controlled On Blur Only

```jsx
// Blur-controlled: shows formatted value, accepts user's raw input while typing
function FormattedCurrencyInput({ value, onChange }) {
  const [displayValue, setDisplayValue] = useState(formatCurrency(value));
  const [isFocused, setIsFocused] = useState(false);

  function handleChange(e) {
    setDisplayValue(e.target.value); // raw input while typing
  }

  function handleFocus() {
    setIsFocused(true);
    setDisplayValue(String(value ?? "")); // show raw number while editing
  }

  function handleBlur() {
    setIsFocused(false);
    const parsed = parseFloat(displayValue.replace(/[^0-9.]/g, ""));
    if (!isNaN(parsed)) {
      onChange(parsed);
      setDisplayValue(formatCurrency(parsed)); // re-format on blur
    } else {
      setDisplayValue(formatCurrency(value)); // restore last valid value
    }
  }

  return (
    <input
      type="text"
      value={displayValue}
      onChange={handleChange}
      onFocus={handleFocus}
      onBlur={handleBlur}
    />
  );
}

function formatCurrency(value) {
  return value == null
    ? ""
    : new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: "USD",
      }).format(value);
}
```

### Compound Controlled Pattern (Upward-Controlled)

```jsx
// Parent controls the state, child is a controlled input
function PriceRangeFilter({ value, onChange }) {
  return (
    <div className="price-range">
      <input
        type="number"
        value={value.min}
        onChange={(e) => onChange({ ...value, min: Number(e.target.value) })}
      />
      <span>to</span>
      <input
        type="number"
        value={value.max}
        onChange={(e) => onChange({ ...value, max: Number(e.target.value) })}
      />
    </div>
  );
}

// Parent:
function Filters() {
  const [priceRange, setPriceRange] = useState({ min: 0, max: 1000 });
  return <PriceRangeFilter value={priceRange} onChange={setPriceRange} />;
}
```

---

## 6. Defaulting and Re-Initializing

```jsx
// defaultValue: sets initial DOM value, ignored on re-renders (uncontrolled)
<input defaultValue="Initial" />  // starts with "Initial", DOM takes over after mount

// value: React controls on every render (controlled)
<input value={controlledValue} onChange={...} />

// ❌ Switching between controlled and uncontrolled causes React warning:
<input value={value ?? ''} /> // if value becomes undefined: controlled → uncontrolled warning

// ✅ Always provide a defined value for controlled inputs
<input value={value ?? ''} onChange={...} /> // never let value become undefined

// Re-initializing a controlled form:
// Simply reset the state variables
function ResetableForm() {
  const [name, setName] = useState('');
  const handleReset = () => setName(''); // controlled reset: trivial
  return (
    <form onReset={handleReset}>
      <input value={name} onChange={e => setName(e.target.value)} />
      <button type="reset">Reset</button>
    </form>
  );
}

// Re-initializing an uncontrolled form (harder):
// Use a key to force remount, or call form.reset()
function ResetableUncontrolledForm({ defaultData }) {
  const formRef = useRef(null);
  const [formKey, setFormKey] = useState(0);

  function handleReset() {
    setFormKey(k => k + 1); // force remount: DOM resets to defaultValue
  }

  return (
    <form key={formKey} ref={formRef}>
      <input name="name" defaultValue={defaultData.name} />
      <button type="button" onClick={handleReset}>Reset</button>
    </form>
  );
}
```

---

## 7. Building a Controlled Form Library Interface

```typescript
// The architecture of a simple controlled form manager
// (simplified version of what Formik does)

interface FormConfig<T extends Record<string, unknown>> {
  initialValues: T;
  onSubmit:      (values: T) => Promise<void> | void;
  validate?:     (values: T) => Partial<Record<keyof T, string>>;
}

function useForm<T extends Record<string, unknown>>(config: FormConfig<T>) {
  const [values,  setValues]  = useState<T>(config.initialValues);
  const [errors,  setErrors]  = useState<Partial<Record<keyof T, string>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const validate = useCallback(() => {
    if (!config.validate) return {};
    return config.validate(values);
  }, [values, config.validate]);

  const handleChange = useCallback(<K extends keyof T>(name: K, value: T[K]) => {
    setValues(prev => ({ ...prev, [name]: value }));
    // Clear error on change
    setErrors(prev => ({ ...prev, [name]: undefined }));
  }, []);

  const handleBlur = useCallback(<K extends keyof T>(name: K) => {
    setTouched(prev => ({ ...prev, [name]: true }));
    const fieldErrors = validate();
    setErrors(fieldErrors);
  }, [validate]);

  const handleSubmit = useCallback(async (e: React.FormEvent) => {
    e.preventDefault();
    const fieldErrors = validate();
    setErrors(fieldErrors);

    // Mark all fields as touched on submit
    const allTouched = Object.fromEntries(
      Object.keys(values).map(k => [k, true])
    ) as Record<keyof T, boolean>;
    setTouched(allTouched);

    if (Object.keys(fieldErrors).length > 0) return;

    setIsSubmitting(true);
    try {
      await config.onSubmit(values);
    } finally {
      setIsSubmitting(false);
    }
  }, [values, validate, config.onSubmit]);

  // Field prop helpers — inject controlled props for an input
  const getFieldProps = useCallback(<K extends keyof T>(name: K) => ({
    value:    values[name] as string,
    onChange: (e: React.ChangeEvent<HTMLInputElement>) =>
      handleChange(name, e.target.value as T[K]),
    onBlur:   () => handleBlur(name),
    name,
  }), [values, handleChange, handleBlur]);

  const getFieldMeta = useCallback(<K extends keyof T>(name: K) => ({
    error:   errors[name],
    touched: touched[name] ?? false,
    showError: !!(touched[name] && errors[name]),
  }), [errors, touched]);

  return { values, errors, touched, isSubmitting, handleSubmit, getFieldProps, getFieldMeta };
}

// Usage:
function SignupForm() {
  const form = useForm({
    initialValues: { email: '', password: '' },
    validate: ({ email, password }) => {
      const errors: Record<string, string> = {};
      if (!email.includes('@')) errors.email = 'Invalid email';
      if (password.length < 8)  errors.password = 'At least 8 characters';
      return errors;
    },
    onSubmit: async (values) => {
      await authApi.signup(values);
    },
  });

  return (
    <form onSubmit={form.handleSubmit}>
      <div>
        <input type="email" {...form.getFieldProps('email')} />
        {form.getFieldMeta('email').showError && (
          <span className="error">{form.getFieldMeta('email').error}</span>
        )}
      </div>
      <div>
        <input type="password" {...form.getFieldProps('password')} />
        {form.getFieldMeta('password').showError && (
          <span className="error">{form.getFieldMeta('password').error}</span>
        )}
      </div>
      <button type="submit" disabled={form.isSubmitting}>Sign Up</button>
    </form>
  );
}
```

---

## 8. React Hook Form — Uncontrolled with Schema

React Hook Form is the leading uncontrolled-first form library. It avoids re-renders by using refs and the native DOM:

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email:    z.string().email('Invalid email'),
  password: z.string().min(8, 'At least 8 characters'),
  age:      z.number().min(18, 'Must be 18+').max(120, 'Invalid age'),
});

type SignupData = z.infer<typeof schema>;

function SignupForm() {
  const {
    register,      // connects inputs to RHF (ref-based, uncontrolled)
    handleSubmit,  // wraps your onSubmit, reads values from DOM/refs
    formState: { errors, isSubmitting },
    watch,         // subscribe to a value (triggers re-render only for that field)
    setValue,      // programmatically set a value
    reset,         // reset form to defaults
  } = useForm<SignupData>({
    resolver: zodResolver(schema), // Zod validation on submit (and optional on change)
    mode:     'onBlur',            // when to validate: 'onChange'|'onBlur'|'onSubmit'
  });

  const onSubmit = async (data: SignupData) => {
    await authApi.signup(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        type="email"
        {...register('email')} // spreads ref + onChange + onBlur + name
      />
      {errors.email && <span>{errors.email.message}</span>}

      <input
        type="password"
        {...register('password')}
      />
      {errors.password && <span>{errors.password.message}</span>}

      <button type="submit" disabled={isSubmitting}>Submit</button>
    </form>
  );
}

// With Controller for custom/third-party components that need controlled behavior:
import { Controller } from 'react-hook-form';

<Controller
  name="country"
  control={control}
  render={({ field }) => (
    <Select
      value={field.value}    // RHF provides controlled interface via Controller
      onChange={field.onChange}
      onBlur={field.onBlur}
      options={countryOptions}
    />
  )}
/>
```

---

## 9. File Inputs and Other Special Cases

```jsx
// File inputs: ALWAYS uncontrolled — you cannot programmatically set a file input's value
function FileUpload({ onUpload }) {
  const fileRef = useRef(null);

  function handleChange(e) {
    const files = Array.from(e.target.files);
    onUpload(files);
  }

  function clear() {
    fileRef.current.value = ""; // clear via ref (not via React state)
  }

  return (
    <>
      <input type="file" ref={fileRef} onChange={handleChange} multiple />
      <button onClick={clear}>Clear</button>
    </>
  );
}

// Checkboxes: use `checked` prop (not `value`) for controlled behavior
function ControlledCheckbox({ checked, onChange }) {
  return (
    <input
      type="checkbox"
      checked={checked} // controlled: React owns the checked state
      onChange={(e) => onChange(e.target.checked)} // passes boolean, not event
    />
  );
}

// Radio buttons: controlled by matching value to the group's state
function RadioGroup({ options, value, onChange }) {
  return options.map((option) => (
    <label key={option.value}>
      <input
        type="radio"
        name="group"
        value={option.value}
        checked={value === option.value} // controlled: checked when values match
        onChange={() => onChange(option.value)}
      />
      {option.label}
    </label>
  ));
}

// Select: use `value` prop for controlled single, `multiple` for multi-select
function ControlledSelect({ value, onChange, options }) {
  return (
    <select value={value} onChange={(e) => onChange(e.target.value)}>
      {options.map((opt) => (
        <option key={opt.value} value={opt.value}>
          {opt.label}
        </option>
      ))}
    </select>
  );
}
```

---

## 10. Controlled Components in Custom UI

```typescript
// The controlled pattern extends beyond HTML inputs to any custom component
// that manages state and should be controllable by the parent

interface SelectProps<T> {
  // Controlled mode:
  value?:       T;
  onChange?:    (value: T) => void;
  // Uncontrolled mode:
  defaultValue?: T;
  // Both modes:
  options:      Array<{ value: T; label: string }>;
}

function Select<T extends string | number>({
  value,
  onChange,
  defaultValue,
  options,
}: SelectProps<T>) {
  // Determine mode: controlled if `value` is explicitly provided
  const isControlled = value !== undefined;

  // Internal state for uncontrolled mode only
  const [internalValue, setInternalValue] = useState<T | undefined>(defaultValue);

  // The "effective" value: controlled value (from parent) or internal state
  const effectiveValue = isControlled ? value : internalValue;

  function handleSelect(newValue: T) {
    if (!isControlled) {
      setInternalValue(newValue); // uncontrolled: update internal state
    }
    onChange?.(newValue); // always notify parent (in case it's semi-controlled)
  }

  return (
    <div className="custom-select">
      {options.map(opt => (
        <div
          key={String(opt.value)}
          className={`option ${opt.value === effectiveValue ? 'selected' : ''}`}
          onClick={() => handleSelect(opt.value)}
        >
          {opt.label}
        </div>
      ))}
    </div>
  );
}

// Usage as controlled:
<Select value={selected} onChange={setSelected} options={options} />

// Usage as uncontrolled (internal state, parent notified):
<Select defaultValue="first" onChange={(v) => console.log('Selected:', v)} options={options} />
```

---

## 11. Good Practices

### ✅ Be explicit: pick controlled OR uncontrolled, don't mix per-field

```jsx
// ✅ Consistent: all fields controlled
const [form, setForm] = useState({ name: '', email: '' });
<input value={form.name} onChange={...} />
<input value={form.email} onChange={...} />

// ❌ Inconsistent: mixing controlled and uncontrolled fields in the same form
<input value={name} onChange={...} />  // controlled
<input ref={emailRef} />               // uncontrolled
```

### ✅ Prevent switching between controlled and uncontrolled

```jsx
// ✅ Always provide a defined value to avoid the controlled→uncontrolled warning
<input value={value ?? ''} onChange={...} />  // never undefined
```

### ✅ Use React Hook Form for forms with many fields or complex validation

```jsx
// ✅ RHF avoids re-renders during typing (critical for 20+ field forms)
const { register, handleSubmit } = useForm();
```

---

## 12. Bad Practices

### ❌ Controlled input with no onChange handler

```jsx
// ❌ Input frozen at initial value — user can't type anything
<input value={name} />
// React controls the DOM value but nothing updates state → input is read-only
// Fix: either add onChange or use defaultValue for uncontrolled

// ✅ Controlled: value + onChange
<input value={name} onChange={e => setName(e.target.value)} />

// ✅ Read-only intention: explicit
<input value={name} readOnly />
```

### ❌ Reading input values from the DOM on submit when controlled

```jsx
// ❌ Unnecessary ref when state already has the value
function Form() {
  const [email, setEmail] = useState("");
  const emailRef = useRef(null);

  function handleSubmit() {
    const emailValue = emailRef.current.value; // unnecessary — email state has it!
  }
  return (
    <input
      value={email}
      onChange={(e) => setEmail(e.target.value)}
      ref={emailRef}
    />
  );
}

// ✅ Use the state value directly
function Form() {
  const [email, setEmail] = useState("");
  function handleSubmit() {
    submitForm({ email }); // use state directly
  }
  return <input value={email} onChange={(e) => setEmail(e.target.value)} />;
}
```

---

## 13. Common Mistakes

### Mistake 1 — Switching between controlled and uncontrolled

```jsx
// ❌ value starts as undefined (uncontrolled) then becomes a string (controlled)
function Component({ initialEmail }) {
  const [email, setEmail] = useState(initialEmail); // undefined on first render!
  return <input value={email} onChange={(e) => setEmail(e.target.value)} />;
  // React warning: "A component is changing an uncontrolled input to be controlled"
}

// ✅ Ensure initial state is always defined
const [email, setEmail] = useState(initialEmail ?? ""); // never undefined
```

### Mistake 2 — Controlled form without instant feedback on slow validation

```jsx
// ❌ Async validation on every keystroke — too slow, UI jank
function EmailInput({ value, onChange }) {
  const [error, setError] = useState("");

  const handleChange = async (e) => {
    onChange(e.target.value);
    const isAvailable = await api.checkEmailAvailability(e.target.value); // async!
    setError(isAvailable ? "" : "Email already taken");
  };
  // Fires an API request on every keystroke — debounce this!
}

// ✅ Debounce async validation
const debouncedCheck = useMemo(
  () =>
    debounce(async (email) => {
      const available = await api.checkEmailAvailability(email);
      setError(available ? "" : "Email already taken");
    }, 500),
  [],
);
```

### Mistake 3 — Using `defaultValue` and expecting it to update

```jsx
// ❌ defaultValue only sets the initial DOM value, ignores prop changes
function UserEditor({ user }) {
  return <input defaultValue={user.name} />; // won't update when user.name changes!
}
// If user.name changes: the input keeps showing the OLD name

// ✅ For updatable external data: use controlled or key-based remount
function UserEditor({ user }) {
  // Option 1: controlled
  const [name, setName] = useState(user.name);
  useEffect(() => setName(user.name), [user.name]); // sync on external change
  return <input value={name} onChange={(e) => setName(e.target.value)} />;

  // Option 2: force remount on user change (simpler, loses focus/cursor position)
  return <input key={user.id} defaultValue={user.name} />;
}
```

---

## 14. Interview-Level Explanation

> **"What's the difference between controlled and uncontrolled components? When would you use each?"**

**Strong answer:**

> "Controlled and uncontrolled refer to who owns the state of an input: React, or the DOM.
>
> In a controlled component, React state is the single source of truth. You set the `value` prop on the input, and every time the user types, an `onChange` handler fires, updates state, React re-renders, and React sets the DOM value to whatever state says it is. The DOM input always reflects React state — you can't have a value in the input that isn't in your state. This gives you full programmatic control: you can validate on every keystroke, transform input as it's typed (like allowing only numbers), or programmatically set values.
>
> In an uncontrolled component, the DOM owns the state. You typically set a `defaultValue` for the initial render and leave the DOM to manage the value independently. When you need the value — on submit, for example — you read it from the DOM via a ref or via `FormData`. No re-renders happen during typing, which is a performance advantage for large forms.
>
> The choice often comes down to what you need to do with the data. If you need real-time validation, dependent fields, or to programmatically change values, controlled is the right choice — it's harder to implement without React owning the state. If you only need the value at submission time and the form is straightforward, uncontrolled is simpler and more performant.
>
> A practical observation: React Hook Form — the most popular form library — is uncontrolled by default. It uses refs and the DOM's native form events to collect values at submit time, only subscribing to individual field state when you explicitly `watch` a field. This dramatically reduces re-renders for large forms. For complex fields that third-party libraries provide (like a date picker or a multi-select dropdown), React Hook Form's `Controller` component wraps them in a controlled interface so they can participate in the form without needing to be natively uncontrolled.
>
> The one common pitfall: switching between controlled and uncontrolled mid-life causes a React warning and can produce bugs. A value that starts as `undefined` makes the input uncontrolled; when state initializes and it becomes a string, you've switched to controlled. The fix is ensuring initial state is always a defined empty value rather than `undefined`."

---

## 15. Exercises

### Exercise 1 — Formatted phone input

Build a controlled phone number input that:

- Formats as the user types: `(555) 123-4567`
- Only allows digits to be entered
- Shows an error if incomplete on blur
- Exposes the raw numeric value (no formatting) to the parent

```typescript
interface PhoneInputProps {
  value: string; // raw digits: "5551234567"
  onChange: (raw: string) => void;
  error?: string;
}
```

<details>
<summary>Solution</summary>

```typescript
function formatPhone(raw: string): string {
  const digits = raw.replace(/\D/g, '').slice(0, 10);
  if (digits.length <= 3) return digits;
  if (digits.length <= 6) return `(${digits.slice(0,3)}) ${digits.slice(3)}`;
  return `(${digits.slice(0,3)}) ${digits.slice(3,6)}-${digits.slice(6)}`;
}

function PhoneInput({ value, onChange, error }: PhoneInputProps) {
  const [displayValue, setDisplayValue] = useState(() => formatPhone(value));
  const [touched, setTouched]           = useState(false);

  // Sync display when external value changes programmatically
  useEffect(() => {
    setDisplayValue(formatPhone(value));
  }, [value]);

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const raw = e.target.value.replace(/\D/g, '').slice(0, 10);
    setDisplayValue(formatPhone(raw));
    onChange(raw); // expose raw digits to parent
  }

  function handleBlur() {
    setTouched(true);
  }

  const hasError = touched && (error || (value.length > 0 && value.length < 10));
  const errorMsg = error ?? (value.length > 0 && value.length < 10 ? 'Enter a complete phone number' : '');

  return (
    <div>
      <input
        type="tel"
        value={displayValue}
        onChange={handleChange}
        onBlur={handleBlur}
        placeholder="(555) 123-4567"
        aria-describedby={hasError ? 'phone-error' : undefined}
        aria-invalid={hasError}
      />
      {hasError && (
        <span id="phone-error" role="alert" className="error">
          {errorMsg}
        </span>
      )}
    </div>
  );
}

// Usage:
function ContactForm() {
  const [phone, setPhone] = useState('');
  return (
    <PhoneInput
      value={phone}
      onChange={setPhone}
      error={phone && phone.length < 10 ? 'Incomplete phone number' : undefined}
    />
  );
}
```

</details>

---

## 🔗 Related Topics

- [`patterns/05-compound-components.md`](./05-compound-components.md) — Advanced component API patterns
- [`patterns/02-custom-hooks.md`](./02-custom-hooks.md) — useForm and similar hook abstractions
- [`testing/02-integration-testing.md`](../testing/02-integration-testing.md) — Testing form behavior

---

<div align="center">

**Next:** [`patterns/05-compound-components.md`](./05-compound-components.md) →

</div>
