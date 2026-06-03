# 04 — State Management Design

> **"State management is not about choosing between Redux, Zustand, and Jotai. It's about deciding which data lives where, who owns it, how it flows, and what happens when it goes stale. The tool is the last decision, not the first."**

State management is one of the most consequential architectural decisions in a frontend application. Get it wrong and you have prop drilling through 10 levels, stale data everywhere, and components that re-render when they shouldn't. Get it right and your app is predictable, testable, and easy to reason about. This document covers the full state taxonomy, design principles for state architecture, patterns that work at scale, and honest tradeoffs between the major tools.

---

## 📚 Table of Contents

1. [The Four Categories of State](#1-the-four-categories-of-state)
2. [State Location Decision Framework](#2-state-location-decision-framework)
3. [Server State — TanStack Query Patterns](#3-server-state--tanstack-query-patterns)
4. [Client State Patterns](#4-client-state-patterns)
5. [URL State](#5-url-state)
6. [Form State](#6-form-state)
7. [State Machines for Complex Flows](#7-state-machines-for-complex-flows)
8. [Global State Architecture](#8-global-state-architecture)
9. [Zustand for Scalable Client State](#9-zustand-for-scalable-client-state)
10. [Atomic State with Jotai](#10-atomic-state-with-jotai)
11. [Redux Toolkit — When It's Right](#11-redux-toolkit--when-its-right)
12. [Derived State and Selectors](#12-derived-state-and-selectors)
13. [Optimistic Updates](#13-optimistic-updates)
14. [Real-Time State Synchronization](#14-real-time-state-synchronization)
15. [Good Practices](#15-good-practices)
16. [Bad Practices](#16-bad-practices)
17. [Common Mistakes](#17-common-mistakes)
18. [Interview-Level Explanation](#18-interview-level-explanation)
19. [Exercises](#19-exercises)

---

## 1. The Four Categories of State

Understanding state categories is the foundation of good state architecture. Each category has different characteristics and should be managed with different tools.

### Category 1 — Server State

```
Definition: Data that originates on the server, cached locally.
Ownership: The server owns the canonical version.
Characteristics:
  - Async (must be fetched)
  - Can be stale (server data may have changed)
  - Can be out of sync across tabs/users
  - Needs loading/error states
  - Benefits from caching and deduplication
  - Needs cache invalidation strategy

Examples:
  User profile, product catalog, order history, notifications
  Anything fetched from an API

Best managed by: TanStack Query, SWR, Apollo Client
```

### Category 2 — Client State (UI State)

```
Definition: State that exists only on the client — not persisted to server.
Ownership: The client owns it entirely.
Characteristics:
  - Synchronous
  - Always up to date (no staleness)
  - No loading/error states
  - Often ephemeral (gone on refresh)

Examples:
  Modal open/closed, active tab, theme preference (if local),
  sidebar collapsed, selected items in a list, animation state

Best managed by: useState, useReducer, Zustand, Jotai
```

### Category 3 — URL State

```
Definition: State encoded in the URL (path, query params, hash).
Ownership: Shared between client and server (server can read it too).
Characteristics:
  - Survives page reload
  - Shareable (send URL to another person)
  - Deep-linkable
  - Tied to browser history (back button)

Examples:
  Search query (?q=...), filters (?status=active&sort=name),
  current page (?page=2), selected tab (/products?tab=reviews),
  modal ID (?modal=delete-confirm&id=42)

Best managed by: React Router, Next.js router, nuqs (URL query params)
```

### Category 4 — Form State

```
Definition: User input values before they're submitted and become server state.
Ownership: Temporary client ownership until submission.
Characteristics:
  - Local to the form component
  - Complex validation needs
  - Dirty tracking (what has the user changed?)
  - Submit handling (loading, error, success)
  - Field-level error messages

Examples:
  Login form, checkout form, settings form, search filters

Best managed by: React Hook Form, Formik (with Hook Form preferred for performance)
```

---

## 2. State Location Decision Framework

```
DECISION TREE:

Where does this data come from?
  Server / API → SERVER STATE → TanStack Query
  User input in a form → FORM STATE → React Hook Form
  Otherwise: continue ↓

Should it survive a page reload?
  Yes → URL STATE (query params, router state)
  No: continue ↓

Is it needed by multiple components far apart in the tree?
  Yes → GLOBAL STATE (Zustand, Jotai, Context)
  No → LOCAL STATE (useState, useReducer)

For global state: how many places need to write to it?
  One writer, many readers → Context + Provider is fine
  Multiple writers, complex updates → Zustand / Redux

How often does it change?
  Every keystroke (forms) → React Hook Form (optimized for this)
  Every few seconds → Local state + polling
  Real-time stream → WebSocket + Zustand/Jotai
```

### State Anti-Pattern Matrix

| Anti-Pattern                      | Problem                              | Fix                        |
| --------------------------------- | ------------------------------------ | -------------------------- |
| Server data in Redux              | Redundant cache, manual invalidation | TanStack Query             |
| All UI state global               | Unnecessary re-renders, coupling     | Move to component useState |
| Form state in Redux               | Re-render on every keystroke         | React Hook Form            |
| Filters/search in component       | Can't deep link, lost on refresh     | URL params                 |
| Navigation state in state manager | Duplicates router state              | React Router useLocation   |

---

## 3. Server State — TanStack Query Patterns

### Query Key Design

```typescript
// Query keys determine cache identity and invalidation
// Design them as hierarchical tuples

// Pattern: ['resource', 'scope', ...filters]
const queryKeys = {
  // Everything about users
  users: () => ["users"] as const,
  // All user lists
  userLists: () => ["users", "list"] as const,
  // Specific user list with filters
  userList: (f: UserFilters) => ["users", "list", f] as const,
  // All user detail records
  userDetails: () => ["users", "detail"] as const,
  // Specific user
  user: (id: string) => ["users", "detail", id] as const,
  // User's orders
  userOrders: (id: string) => ["users", id, "orders"] as const,
};

// Invalidation power:
// Invalidate everything:          queryClient.invalidateQueries({ queryKey: queryKeys.users() })
// Invalidate all lists:           queryClient.invalidateQueries({ queryKey: queryKeys.userLists() })
// Invalidate one user:            queryClient.invalidateQueries({ queryKey: queryKeys.user('42') })
// Invalidate only specific list:  queryClient.invalidateQueries({ queryKey: queryKeys.userList({...}) })
```

### Mutation with Optimistic Updates

```typescript
function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: UpdateUserData) => usersApi.update(data.id, data),

    onMutate: async (data) => {
      // 1. Cancel in-flight queries for this user (avoid race conditions)
      await queryClient.cancelQueries({ queryKey: queryKeys.user(data.id) });

      // 2. Snapshot the previous value for rollback
      const previousUser = queryClient.getQueryData(queryKeys.user(data.id));

      // 3. Optimistically update the cache
      queryClient.setQueryData(queryKeys.user(data.id), (old: User) => ({
        ...old,
        ...data,
      }));

      // 4. Return context for rollback
      return { previousUser };
    },

    onError: (err, data, context) => {
      // Rollback: restore previous value on error
      queryClient.setQueryData(queryKeys.user(data.id), context?.previousUser);
    },

    onSettled: (data, err, variables) => {
      // Always refetch after mutation to ensure consistency
      queryClient.invalidateQueries({ queryKey: queryKeys.user(variables.id) });
    },
  });
}
```

### Query Prefetching

```typescript
// Prefetch on hover — data ready before navigation
function ProductCard({ productId }: { productId: string }) {
  const queryClient = useQueryClient();

  const prefetchProduct = useCallback(() => {
    queryClient.prefetchQuery({
      queryKey: queryKeys.product(productId),
      queryFn:  () => productsApi.get(productId),
      staleTime: 30_000, // don't refetch if cached within 30s
    });
  }, [productId, queryClient]);

  return (
    <div onMouseEnter={prefetchProduct}>
      {/* ... */}
    </div>
  );
}
```

### Dependent Queries

```typescript
// Query that depends on another query's result
function UserOrdersView({ userId }: { userId: string }) {
  // First: fetch user
  const { data: user } = useQuery({
    queryKey: queryKeys.user(userId),
    queryFn:  () => usersApi.get(userId),
  });

  // Second: fetch orders only when user is available
  const { data: orders } = useQuery({
    queryKey: queryKeys.userOrders(userId),
    queryFn:  () => ordersApi.getForUser(userId),
    enabled:  !!user,                  // only runs when user is loaded
    staleTime: 60_000,
  });

  if (!user || !orders) return <Skeleton />;
  return <OrderList user={user} orders={orders} />;
}
```

---

## 4. Client State Patterns

### useReducer for Complex Local State

```typescript
// Use useReducer when: multiple related state values,
// complex state transitions, or state depends on previous state

type CartState = {
  items: CartItem[];
  loading: boolean;
  error: string | null;
};

type CartAction =
  | { type: "ADD_ITEM"; payload: CartItem }
  | { type: "REMOVE_ITEM"; payload: string } // id
  | { type: "UPDATE_QTY"; payload: { id: string; qty: number } }
  | { type: "CLEAR" }
  | { type: "LOAD_START" }
  | { type: "LOAD_SUCCESS"; payload: CartItem[] }
  | { type: "LOAD_ERROR"; payload: string };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM":
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map((i) =>
            i.id === action.payload.id
              ? { ...i, quantity: i.quantity + action.payload.quantity }
              : i,
          ),
        };
      }
      return { ...state, items: [...state.items, action.payload] };

    case "REMOVE_ITEM":
      return {
        ...state,
        items: state.items.filter((i) => i.id !== action.payload),
      };

    case "UPDATE_QTY":
      return {
        ...state,
        items: state.items
          .map((i) =>
            i.id === action.payload.id
              ? { ...i, quantity: action.payload.qty }
              : i,
          )
          .filter((i) => i.quantity > 0), // remove if qty reaches 0
      };

    case "CLEAR":
      return { ...state, items: [] };
    case "LOAD_START":
      return { ...state, loading: true, error: null };
    case "LOAD_SUCCESS":
      return { ...state, loading: false, items: action.payload };
    case "LOAD_ERROR":
      return { ...state, loading: false, error: action.payload };

    default:
      return state;
  }
}

function useCartReducer() {
  return useReducer(cartReducer, { items: [], loading: false, error: null });
}
```

---

## 5. URL State

URL state is underutilized — it provides deep-linking, shareability, and browser history for free.

### What Belongs in URL State

```typescript
// ✅ Good URL state candidates:
?q=leather+boots          // search query
?category=footwear&brand=nike // filter state
?sort=price&dir=asc        // sort preferences
?page=3                    // pagination
?tab=reviews               // active tab
?modal=size-guide&product=42 // modal state
?view=grid                 // layout preference

// ❌ Bad URL state candidates:
?animation=running         // ephemeral UI state
?hover=productId           // too transient
?scroll=1240               // scroll position (unreliable)
```

### URL State with nuqs

```typescript
import { useQueryState, parseAsInteger, parseAsString } from 'nuqs';

function ProductsFilters() {
  // Type-safe URL query params
  const [search,   setSearch]   = useQueryState('q',        parseAsString.withDefault(''));
  const [page,     setPage]     = useQueryState('page',     parseAsInteger.withDefault(1));
  const [category, setCategory] = useQueryState('category', parseAsString.withDefault('all'));
  const [sortBy,   setSortBy]   = useQueryState('sort',     parseAsString.withDefault('relevance'));

  // Any change to these updates the URL (shareable, bookmarkable)
  // Browser back button restores previous filter state

  return (
    <div>
      <SearchInput
        value={search}
        onChange={q => { setSearch(q); setPage(1); }} // reset page on new search
      />
      <CategorySelect
        value={category}
        onChange={setCategory}
      />
      <SortSelect
        value={sortBy}
        onChange={setSortBy}
      />
    </div>
  );
}
```

### Syncing URL State with Server State

```typescript
function useProductSearch() {
  const [search] = useQueryState("q", parseAsString.withDefault(""));
  const [category] = useQueryState("cat", parseAsString.withDefault("all"));
  const [page] = useQueryState("page", parseAsInteger.withDefault(1));

  // URL params become query keys — URL changes trigger refetch
  return useQuery({
    queryKey: ["products", "search", { search, category, page }],
    queryFn: () => productsApi.search({ search, category, page }),
    placeholderData: keepPreviousData, // keep previous results during navigation
  });
}
```

---

## 6. Form State

### React Hook Form — The Performance-Correct Approach

```typescript
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Schema-first validation
const checkoutSchema = z.object({
  email:     z.string().email('Invalid email'),
  firstName: z.string().min(1, 'Required').max(50),
  lastName:  z.string().min(1, 'Required').max(50),
  address:   z.object({
    street:  z.string().min(1, 'Required'),
    city:    z.string().min(1, 'Required'),
    zip:     z.string().regex(/^\d{5}(-\d{4})?$/, 'Invalid ZIP code'),
    country: z.string().min(2, 'Required'),
  }),
  saveAddress: z.boolean().default(false),
});

type CheckoutFormData = z.infer<typeof checkoutSchema>;

function CheckoutForm({ onSubmit }: { onSubmit: (data: CheckoutFormData) => Promise<void> }) {
  const {
    register,
    handleSubmit,
    control,
    formState: { errors, isSubmitting, isDirty, touchedFields },
    reset,
    setError,
  } = useForm<CheckoutFormData>({
    resolver: zodResolver(checkoutSchema),
    defaultValues: {
      email:     '',
      firstName: '',
      address:   { country: 'US' },
      saveAddress: false,
    },
    mode: 'onBlur', // validate on blur, not on every keystroke
  });

  const submitHandler = handleSubmit(async (data) => {
    try {
      await onSubmit(data);
      reset(); // clear form on success
    } catch (error) {
      // Server validation errors
      if (isServerValidationError(error)) {
        error.fields.forEach(({ field, message }) => {
          setError(field as keyof CheckoutFormData, { message });
        });
      }
    }
  });

  return (
    <form onSubmit={submitHandler}>
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          {...register('email')} // uncontrolled — no re-render on type
        />
        {errors.email && <span role="alert">{errors.email.message}</span>}
      </div>

      {/* Controlled input when you need custom UI */}
      <Controller
        name="address.country"
        control={control}
        render={({ field, fieldState }) => (
          <CountrySelect
            value={field.value}
            onChange={field.onChange}
            error={fieldState.error?.message}
          />
        )}
      />

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Placing Order...' : 'Place Order'}
      </button>
    </form>
  );
}
```

---

## 7. State Machines for Complex Flows

Complex multi-step flows (checkout, onboarding, upload) benefit from explicit state machines.

```typescript
// XState v5 for explicit flow management
import { createMachine, assign } from 'xstate';

type CheckoutContext = {
  cart:     CartItem[];
  shipping: ShippingData | null;
  payment:  PaymentData | null;
  orderId:  string | null;
  error:    string | null;
};

const checkoutMachine = createMachine({
  id: 'checkout',
  initial: 'cart',
  types: {} as { context: CheckoutContext },

  context: {
    cart:     [],
    shipping: null,
    payment:  null,
    orderId:  null,
    error:    null,
  },

  states: {
    cart: {
      on: {
        PROCEED_TO_SHIPPING: {
          guard: 'hasItems',
          target: 'shipping',
        },
      },
    },
    shipping: {
      on: {
        SUBMIT_SHIPPING: {
          target: 'payment',
          actions: assign({ shipping: ({ event }) => event.data }),
        },
        BACK: 'cart',
      },
    },
    payment: {
      on: {
        SUBMIT_PAYMENT: {
          target: 'placing-order',
          actions: assign({ payment: ({ event }) => event.data }),
        },
        BACK: 'shipping',
      },
    },
    'placing-order': {
      invoke: {
        src: 'placeOrder',
        input: ({ context }) => ({
          cart:     context.cart,
          shipping: context.shipping!,
          payment:  context.payment!,
        }),
        onDone: {
          target: 'confirmation',
          actions: assign({ orderId: ({ event }) => event.output.orderId }),
        },
        onError: {
          target: 'payment',
          actions: assign({ error: ({ event }) => event.error.message }),
        },
      },
    },
    confirmation: { type: 'final' },
  },
}, {
  guards: {
    hasItems: ({ context }) => context.cart.length > 0,
  },
});

// Usage in React
function CheckoutFlow() {
  const [state, send] = useMachine(checkoutMachine, {
    actors: {
      placeOrder: fromPromise(({ input }) => checkoutApi.place(input)),
    },
  });

  if (state.matches('cart'))          return <CartReview onProceed={() => send({ type: 'PROCEED_TO_SHIPPING' })} />;
  if (state.matches('shipping'))      return <ShippingForm onSubmit={(data) => send({ type: 'SUBMIT_SHIPPING', data })} />;
  if (state.matches('payment'))       return <PaymentForm error={state.context.error} />;
  if (state.matches('placing-order')) return <ProcessingOrder />;
  if (state.matches('confirmation'))  return <OrderConfirmation orderId={state.context.orderId!} />;
}
```

---

## 8. Global State Architecture

### When Global State Is Justified

```
Global state makes sense when:
  ✓ Multiple features need the same data
  ✓ Multiple components far apart in the tree write to it
  ✓ State must survive component unmount/remount
  ✓ State is genuinely application-wide (auth, theme, notifications)

Global state is overkill for:
  ✗ State needed by only one component (use local state)
  ✗ State needed by a subtree (use Context for that subtree)
  ✗ Server data (use TanStack Query)
  ✗ Form data (use React Hook Form)
```

### Sliced Global State

```typescript
// Divide global state into independent slices — separate concerns
// Don't have one massive global store

// auth-store.ts
interface AuthStore {
  user: User | null;
  token: string | null;
  isReady: boolean;
  setUser: (user: User, token: string) => void;
  clearAuth: () => void;
}

// ui-store.ts
interface UIStore {
  theme: "light" | "dark";
  sidebarOpen: boolean;
  setTheme: (theme: "light" | "dark") => void;
  toggleSidebar: () => void;
}

// notifications-store.ts
interface NotificationsStore {
  items: Notification[];
  addToast: (n: Notification) => void;
  removeToast: (id: string) => void;
}

// Each slice is independent — components subscribe only to what they need
```

---

## 9. Zustand for Scalable Client State

Zustand is the current sweet spot for client global state: minimal boilerplate, no Provider needed, excellent TypeScript support.

```typescript
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';
import { devtools, persist }     from 'zustand/middleware';
import { immer }                  from 'zustand/middleware/immer';

interface AuthStore {
  user:    User | null;
  token:   string | null;
  isReady: boolean;

  // Actions
  setAuth:    (user: User, token: string) => void;
  clearAuth:  () => void;
  updateUser: (partial: Partial<User>) => void;
}

export const useAuthStore = create<AuthStore>()(
  devtools(      // Redux DevTools support
    persist(     // persists to localStorage
      immer(     // immer for mutable update syntax
        subscribeWithSelector((set) => ({
          user:    null,
          token:   null,
          isReady: false,

          setAuth: (user, token) => set(state => {
            state.user    = user;
            state.token   = token;
            state.isReady = true;
          }),

          clearAuth: () => set(state => {
            state.user    = null;
            state.token   = null;
            state.isReady = true;
          }),

          updateUser: (partial) => set(state => {
            if (state.user) Object.assign(state.user, partial);
          }),
        }))
      ),
      {
        name:      'auth-store',
        // Only persist what makes sense
        partialize: (state) => ({ user: state.user, token: state.token }),
      }
    )
  )
);

// Selective subscription — only re-render when user changes
function UserAvatar() {
  const user = useAuthStore(state => state.user); // ← only user
  return user ? <img src={user.avatarUrl} alt={user.name} /> : null;
}

// Outside React (in services, workers)
const token = useAuthStore.getState().token;
useAuthStore.subscribe(state => state.token, (token) => {
  apiClient.defaults.headers.Authorization = `Bearer ${token}`;
});
```

### Zustand Slices for Large Stores

```typescript
// For large stores: compose slices
type BoundStore = AuthSlice & UISlice & CartSlice;

const useStore = create<BoundStore>()((...a) => ({
  ...createAuthSlice(...a),
  ...createUISlice(...a),
  ...createCartSlice(...a),
}));

// Each slice:
type AuthSlice = { user: User | null; setUser: (u: User) => void };
const createAuthSlice: StateCreator<BoundStore, [], [], AuthSlice> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
});
```

---

## 10. Atomic State with Jotai

Jotai models state as atoms — fine-grained reactive units. Excellent for complex derived state and when performance is critical.

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';
import { atomWithStorage, loadable }               from 'jotai/utils';

// Primitive atoms
const userAtom      = atom<User | null>(null);
const themeAtom     = atomWithStorage<'light' | 'dark'>('theme', 'light');
const cartItemsAtom = atom<CartItem[]>([]);

// Derived atoms (computed from other atoms, no redundant state)
const cartCountAtom  = atom((get) => get(cartItemsAtom).length);
const cartTotalAtom  = atom((get) =>
  get(cartItemsAtom).reduce((sum, item) => sum + item.price * item.quantity, 0)
);
const isLoggedInAtom = atom((get) => get(userAtom) !== null);

// Write-only atom (action)
const addToCartAtom = atom(
  null, // read: null (write-only)
  (get, set, item: CartItem) => {
    const existing = get(cartItemsAtom).find(i => i.id === item.id);
    if (existing) {
      set(cartItemsAtom, get(cartItemsAtom).map(i =>
        i.id === item.id ? { ...i, quantity: i.quantity + item.quantity } : i
      ));
    } else {
      set(cartItemsAtom, [...get(cartItemsAtom), item]);
    }
  }
);

// Async atom with loadable
const userDataAtom = atom(async (get) => {
  const user = get(userAtom);
  if (!user) return null;
  return fetchUserData(user.id);
});

// Usage — components only re-render when their specific atoms change
function CartBadge() {
  const count = useAtomValue(cartCountAtom); // only re-renders when count changes
  return <span>{count}</span>;
}

function CartTotal() {
  const total = useAtomValue(cartTotalAtom); // only re-renders when total changes
  return <span>${total.toFixed(2)}</span>;
}

function AddToCartButton({ item }: { item: CartItem }) {
  const addToCart = useSetAtom(addToCartAtom); // no re-render on cart changes
  return <button onClick={() => addToCart(item)}>Add to Cart</button>;
}
```

---

## 11. Redux Toolkit — When It's Right

Redux Toolkit (RTK) is excellent for complex state with many interactions, strong DevTools needs, and teams that benefit from strict conventions.

```typescript
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

// Async thunk
export const fetchUsers = createAsyncThunk(
  "users/fetchAll",
  async (filters: UserFilters, { rejectWithValue }) => {
    try {
      return await usersApi.list(filters);
    } catch (err) {
      return rejectWithValue(err.message);
    }
  },
);

const usersSlice = createSlice({
  name: "users",
  initialState: {
    items: [] as User[],
    loading: false,
    error: null as string | null,
    selected: null as string | null,
  },
  reducers: {
    selectUser: (state, action: PayloadAction<string>) => {
      state.selected = action.payload; // immer handles immutability
    },
    clearSelection: (state) => {
      state.selected = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload as string;
      });
  },
});

export const { selectUser, clearSelection } = usersSlice.actions;
export default usersSlice.reducer;
```

### When to Choose Redux vs Zustand

```
Choose Redux Toolkit when:
  ✓ Complex state interactions across many slices (Redux DevTools invaluable)
  ✓ Large team that benefits from enforced conventions
  ✓ Time-travel debugging is important
  ✓ Already using Redux and RTK is an upgrade
  ✓ Need middleware (RTK Query, custom analytics middleware)

Choose Zustand when:
  ✓ Smaller to medium codebase
  ✓ Want minimal boilerplate
  ✓ Outside React (services, workers) needs store access
  ✓ Flexibility is more important than convention enforcement
  ✓ Starting fresh — Zustand is simpler to get right

Choose Jotai when:
  ✓ Fine-grained reactivity matters (minimize re-renders)
  ✓ Complex derived/computed state
  ✓ Server state mixed with client state (async atoms)
  ✓ Code splitting concerns (atoms are naturally code-split)
```

---

## 12. Derived State and Selectors

Never store derived data — compute it from canonical state.

```typescript
// ❌ Storing derived state (redundant, can go stale)
const cartStore = {
  items: [],
  totalPrice: 0, // derived from items — stays stale!
  itemCount: 0, // derived from items — stays stale!
  hasDiscount: false, // derived from items/promo — stays stale!
};

// ✅ Derive on read (memoized for performance)
const cartStore = {
  items: [],
  promoCode: null,
};

// Selectors — computed from canonical state
const selectItemCount = (state: CartState) =>
  state.items.reduce((sum, item) => sum + item.quantity, 0);

const selectTotalPrice = (state: CartState) => {
  const subtotal = state.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0,
  );
  const discount = state.promoCode ? getDiscount(state.promoCode, subtotal) : 0;
  return subtotal - discount;
};

// With reselect (memoized)
const selectExpensiveItems = createSelector(
  [(state: RootState) => state.cart.items],
  (items) => items.filter((item) => item.price > 100),
);
```

### Zustand Computed Values

```typescript
// With zustand: compute in selector passed to useStore
const useCartTotal = () =>
  useCartStore(
    useShallow((state) => ({
      total: state.items.reduce((sum, i) => sum + i.price * i.quantity, 0),
      count: state.items.reduce((sum, i) => sum + i.quantity, 0),
      isEmpty: state.items.length === 0,
    })),
  );
```

---

## 13. Optimistic Updates

Show the result immediately, then confirm with the server.

```typescript
function useToggleTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (todo: Todo) =>
      todosApi.update(todo.id, { completed: !todo.completed }),

    onMutate: async (todo) => {
      // Cancel any refetches
      await queryClient.cancelQueries({ queryKey: ["todos"] });

      // Snapshot
      const previous = queryClient.getQueryData<Todo[]>(["todos"]);

      // Optimistic update — immediate UI response
      queryClient.setQueryData<Todo[]>(["todos"], (old = []) =>
        old.map((t) =>
          t.id === todo.id ? { ...t, completed: !t.completed } : t,
        ),
      );

      // Return snapshot for potential rollback
      return { previous };
    },

    onError: (err, todo, context) => {
      // Rollback on error
      if (context?.previous) {
        queryClient.setQueryData(["todos"], context.previous);
      }
      // Notify user
      toast.error("Failed to update todo. Please try again.");
    },

    onSettled: () => {
      // Always refetch to ensure consistency
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });
}
```

---

## 14. Real-Time State Synchronization

```typescript
// WebSocket + Zustand for real-time updates
class RealtimeSync {
  #socket: WebSocket | null = null;
  #cleanup: (() => void)[] = [];

  connect(userId: string) {
    this.#socket = new WebSocket(
      `wss://api.example.com/realtime?userId=${userId}`,
    );

    this.#socket.onmessage = ({ data }) => {
      const event = JSON.parse(data);
      this.#handleEvent(event);
    };

    this.#socket.onclose = () => {
      // Reconnect with exponential backoff
      setTimeout(() => this.connect(userId), 2000);
    };
  }

  #handleEvent(event: RealtimeEvent) {
    const store = useAppStore.getState();

    switch (event.type) {
      case "notification:new":
        store.addNotification(event.payload);
        break;

      case "cart:updated-by-other-device":
        // Invalidate TanStack Query cache
        queryClient.invalidateQueries({ queryKey: ["cart"] });
        break;

      case "user:profile-updated":
        store.updateUser(event.payload);
        break;
    }
  }

  disconnect() {
    this.#socket?.close();
    this.#cleanup.forEach((fn) => fn());
  }
}
```

---

## 15. Good Practices

### ✅ Co-locate state with its consumers

```typescript
// ✅ If only one component needs this state: keep it local
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false); // local — correct
  // ...
}

// ✅ If a feature needs it: keep it in the feature
function useCartFeature() {
  const [selectedShipping, setSelectedShipping] = useState<string | null>(null);
  // ...
}
```

### ✅ Keep TanStack Query as the single source of truth for server data

```typescript
// ✅ Don't copy server data to Zustand store
// If user profile is in TanStack Query cache: read it from there
const { data: user } = useQuery({ queryKey: ["user"], queryFn: fetchUser });

// ❌ Don't do this: copying server data to global store
useEffect(() => {
  if (user) zustandStore.setUser(user); // TanStack Query IS your cache
}, [user]);
```

### ✅ Make impossible states impossible

```typescript
// ❌ These combinations are invalid but possible:
const [isLoading, setIsLoading] = useState(false);
const [data, setData] = useState<User | null>(null);
const [error, setError] = useState<Error | null>(null);
// isLoading=true, data=SomeUser, error=SomeError — impossible but representable

// ✅ Discriminated union (one of four states, mutually exclusive)
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

const [state, setState] = useState<AsyncState<User>>({ status: "idle" });
```

---

## 16. Bad Practices

### ❌ Using global state for everything

```typescript
// ❌ Global Redux store for modal state
dispatch(openModal({ id: "confirm-delete" }));
dispatch(closeModal({ id: "confirm-delete" }));
// This couples every component to the same store shape

// ✅ Modal state local to the triggering component
const [showDeleteConfirm, setShowDeleteConfirm] = useState(false);
```

### ❌ Over-fetching and ignoring cache

```typescript
// ❌ Fetching the same user on every component mount
function UserAvatar({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    fetchUser(userId).then(setUser); // N components = N fetches
  }, [userId]);
}

// ✅ TanStack Query deduplicates concurrent requests
function UserAvatar({ userId }: { userId: string }) {
  const { data: user } = useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,
  });
  // 100 components with same userId → 1 fetch, shared cache
}
```

### ❌ Putting async operations directly in reducers

```typescript
// ❌ Async in reducer — pure functions only
const reducer = (state, action) => {
  if (action.type === "FETCH_USER") {
    fetchUser(action.id).then((user) => dispatch({ type: "SET_USER", user }));
    // Mutates externally in a "pure" function — unpredictable
  }
};

// ✅ Async in thunks, sagas, or TanStack Query
const fetchUserThunk = createAsyncThunk("users/fetch", async (id) =>
  usersApi.get(id),
);
```

---

## 17. Common Mistakes

### Mistake 1 — Not memoizing context values

```typescript
// ❌ New object on every render → all context consumers re-render
function AuthProvider({ children }) {
  const [user, setUser] = useState<User | null>(null);

  return (
    <AuthContext.Provider value={{ user, setUser }}> {/* new object every render! */}
      {children}
    </AuthContext.Provider>
  );
}

// ✅ Memoize the context value
function AuthProvider({ children }) {
  const [user, setUser] = useState<User | null>(null);

  const value = useMemo(() => ({ user, setUser }), [user]);

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}
```

### Mistake 2 — Conflating loading states

```typescript
// ❌ One loading state for all operations — confusing
const [loading, setLoading] = useState(false);
// Is it loading the initial data? Saving? Deleting?

// ✅ Separate loading states per operation
const { isLoading: isFetching }   = useQuery(...);
const { isPending: isSaving }     = useMutation(...);
const { isPending: isDeleting }   = useMutation(...);
```

### Mistake 3 — Syncing state instead of deriving it

```typescript
// ❌ Syncing: whenever products change, update filteredProducts
const [products, setProducts] = useState<Product[]>([]);
const [filteredProducts, setFilteredProducts] = useState<Product[]>([]);
const [filter, setFilter] = useState("");

useEffect(() => {
  setFilteredProducts(products.filter((p) => p.name.includes(filter)));
}, [products, filter]); // sync: easy to get stale

// ✅ Derive: compute filteredProducts from products + filter
const filteredProducts = useMemo(
  () => products.filter((p) => p.name.includes(filter)),
  [products, filter],
);
// Always consistent — can never be stale
```

---

## 18. Interview-Level Explanation

> **"How do you approach state management in a large React application?"**

**Strong answer:**

> "I start by categorizing state rather than picking a tool. There are four types with distinct characteristics.
>
> Server state — data from APIs — is the most commonly mismanaged. It's async, can be stale, and needs caching. TanStack Query handles this perfectly: it caches by query key, deduplicates concurrent requests, handles loading/error states, and provides background refetching. Putting API data in Redux was the standard approach five years ago; TanStack Query made that pattern largely obsolete.
>
> Client state — UI state that never reaches the server — usually doesn't need to be global. A modal's open/closed state, an active tab, a tooltip's visibility: these should be local `useState` in the component that controls them. Most applications have far less genuinely global client state than developers think. For what is truly global — auth, theme, shopping cart — I use Zustand for its minimal boilerplate and ability to be read outside React components.
>
> URL state is underused. Filters, search queries, pagination, active tabs — these should live in the URL so users can share links and use the back button naturally. `nuqs` makes this type-safe and clean.
>
> Form state is its own category. React Hook Form handles it perfectly because it's uncontrolled by default — no re-renders on every keystroke. It provides field-level validation, dirty tracking, and submit handling without touching your global store.
>
> The principle that ties everything together: don't duplicate state. Derived data should be computed, not stored. Selectors compute totals from cart items instead of maintaining a separate `totalPrice` field that can go stale. TanStack Query is the cache for server data — don't copy it into Zustand. URL is the source of truth for shareable state — don't mirror it in component state. One source of truth per piece of data is the invariant that prevents the entire class of 'stale state' bugs."

---

## 19. Exercises

### Exercise 1 — Classify this state

For each piece of state, identify: (1) which category it belongs to, (2) the right tool, (3) where it should live.

```
a) The currently logged-in user's profile
b) Whether the "delete account" confirmation modal is open
c) A list of products fetched from /api/products
d) The current search query in the products page
e) Form field values for the checkout address form
f) The user's dark/light theme preference (persists across sessions)
g) A list of notifications loaded from the server
h) Whether a specific product card is showing its hover tooltip
i) The current checkout step (1 of 3) in a multi-step form
j) The filters applied to the products page (category, price range)
```

<details>
<summary>Answers</summary>

```
a) User profile → SERVER STATE (from API, async, can be stale)
   Tool: TanStack Query, queryKey: ['user', 'me']
   Location: TanStack Query cache, accessed via useQuery hook

b) Delete modal open/closed → CLIENT UI STATE (ephemeral, no server)
   Tool: useState
   Location: Component that owns the modal (not global)

c) Products list → SERVER STATE (from API)
   Tool: TanStack Query, queryKey: ['products', filters]
   Location: TanStack Query cache

d) Search query → URL STATE (shareable, browser history)
   Tool: nuqs useQueryState or React Router useSearchParams
   Location: URL ?q=search-term

e) Checkout form fields → FORM STATE (temp before submit)
   Tool: React Hook Form
   Location: CheckoutForm component (not global)

f) Theme preference (persisted) → CLIENT STATE + PERSISTENCE
   Tool: Zustand with persist middleware or atomWithStorage (Jotai)
   Location: Global store (cross-app concern)

g) Notifications list → SERVER STATE (from API, real-time updates)
   Tool: TanStack Query + WebSocket invalidation
   Location: TanStack Query cache

h) Product card hover tooltip → CLIENT UI STATE (ephemeral, local)
   Tool: useState
   Location: ProductCard component (very local)

i) Checkout step → CLIENT STATE (could be URL for deep-linking)
   Tool: useState for simple flow, or URL state for shareable steps
   If complex: state machine (XState)
   Location: CheckoutFlow component or URL (/checkout/shipping)

j) Product filters → URL STATE (shareable, browser history)
   Tool: nuqs, URL params: ?category=electronics&min=50&max=200
   Location: URL — feeds into TanStack Query queryKey for server fetch
```

</details>

---

## 🔗 Related Topics

- [`system-design/01-large-scale-architecture.md`](./01-large-scale-architecture.md) — Architecture that shapes state design
- [`system-design/02-feature-based-structure.md`](./02-feature-based-structure.md) — State location within feature modules
- [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md) — Async patterns for server state
- [`performance/07-memoization.md`](../performance/07-memoization.md) — Memoizing derived state

---

<div align="center">

**Next:** [`system-design/05-config-driven-ui.md`](./05-config-driven-ui.md) →

</div>
