# 04 — Data Caching

> **"Data caching is not about making your app faster — it's about making your app feel instant. The network is slow and unpredictable. Data caching is what stands between the user and that reality, serving yesterday's response today while quietly fetching tomorrow's."**

Data caching sits between memory caching (nanoseconds, volatile) and HTTP caching (persistent, header-driven). It's the application-level decision layer: which data to cache, how to keep it fresh, how to handle optimistic updates, how to invalidate across features, and how to survive cache invalidation bugs without breaking users. This document covers TanStack Query patterns, IndexedDB for persistent data, optimistic mutation design, cache synchronization, and offline-first data architecture.

---

## 📚 Table of Contents

1. [The Data Caching Problem Space](#1-the-data-caching-problem-space)
2. [TanStack Query — The Gold Standard](#2-tanstack-query--the-gold-standard)
3. [Query Key Design for Cache Control](#3-query-key-design-for-cache-control)
4. [Cache Invalidation Patterns](#4-cache-invalidation-patterns)
5. [Optimistic Mutations with Rollback](#5-optimistic-mutations-with-rollback)
6. [IndexedDB for Persistent Data Cache](#6-indexeddb-for-persistent-data-cache)
7. [Cache Synchronization Across Tabs](#7-cache-synchronization-across-tabs)
8. [Offline-First Data Architecture](#8-offline-first-data-architecture)
9. [Real-Time Cache Updates via WebSocket](#9-real-time-cache-updates-via-websocket)
10. [Cache Warming Strategies](#10-cache-warming-strategies)
11. [Cache Debugging and Inspection](#11-cache-debugging-and-inspection)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Data Caching Problem Space

```
THE PROBLEMS DATA CACHING SOLVES:

1. LATENCY:
   Every API call: 100-500ms round trip
   Without cache: every component mount = network wait
   With cache: instant (serve previous data, refetch in background)

2. DEDUPLICATION:
   10 components need the same user profile simultaneously
   Without cache: 10 parallel network requests for the same data
   With cache: 1 request, shared by all 10 components

3. BACKGROUND FRESHNESS:
   Data changes while user is on the page
   Without cache: user sees stale data forever
   With cache + revalidation: data refreshes in background automatically

4. OPTIMISTIC UI:
   User clicks "Like" — must it show instantly or after server confirms?
   With optimistic cache: show result instantly, confirm/rollback after server responds

5. OFFLINE SUPPORT:
   Network drops during critical flow
   With cache: serve last known data, queue mutations for retry
```

### Data Cache vs Memory Cache vs HTTP Cache

```
HTTP CACHE (browser disk):
  Scope:       single browser, persistent
  Controlled by: response headers
  Best for:    static assets, public API responses
  Invalidation: URL change, CDN purge, max-age expiry

MEMORY CACHE (JavaScript):
  Scope:       single tab, lost on refresh
  Controlled by: your code (LRU, TTL)
  Best for:    derived computations, expensive function results
  Invalidation: explicit delete, TTL, LRU eviction

DATA CACHE (TanStack Query, SWR, Apollo):
  Scope:       application session, with optional persistence
  Controlled by: query keys, staleTime, gcTime
  Best for:    server state (API responses)
  Invalidation: explicit invalidation, refetch, mutation
```

---

## 2. TanStack Query — The Gold Standard

TanStack Query provides declarative server state management with built-in caching, deduplication, background refresh, and mutation handling.

### Query Configuration Deep Dive

```typescript
useQuery({
  // IDENTITY: what data this query fetches
  queryKey: ["products", { category: "electronics", page: 1 }],
  queryFn: () => productsApi.list({ category: "electronics", page: 1 }),

  // FRESHNESS
  staleTime: 5 * 60_000, // consider fresh for 5 minutes (no background refetch)
  gcTime: 30 * 60_000, // keep in cache for 30 minutes after all subscribers unmount
  // (gcTime was formerly called cacheTime)

  // REFETCH BEHAVIOR
  refetchOnMount: true, // refetch when component mounts (if stale)
  refetchOnWindowFocus: true, // refetch when tab regains focus (if stale)
  refetchOnReconnect: true, // refetch when network reconnects (if stale)
  refetchInterval: false, // polling interval (false = no polling)
  refetchIntervalInBackground: false,

  // CONDITIONAL
  enabled: !!userId, // only fetch when userId is truthy

  // DATA TRANSFORMATION
  select: (data) => ({
    items: data.results,
    totalCount: data.meta.total,
    hasMore: data.meta.hasMore,
  }),

  // PLACEHOLDERS
  placeholderData: keepPreviousData, // show previous data during page transitions
  initialData: () => ({ items: [], totalCount: 0 }), // synchronous initial data

  // ERROR HANDLING
  retry: 3, // retry 3 times on failure
  retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30_000), // exponential backoff

  // NETWORK
  networkMode: "online", // 'online' | 'always' | 'offlineFirst'
});
```

### Parallel and Dependent Queries

```typescript
// Parallel queries: run simultaneously
function UserDashboard({ userId }: { userId: string }) {
  const userQuery = useQuery({
    queryKey: ["user", userId],
    queryFn: () => getUser(userId),
  });
  const ordersQuery = useQuery({
    queryKey: ["orders", userId],
    queryFn: () => getOrders(userId),
  });
  const reviewsQuery = useQuery({
    queryKey: ["reviews", userId],
    queryFn: () => getReviews(userId),
  });
  // All three fire simultaneously — no waterfall

  // Or use useQueries for dynamic parallel queries:
  const productQueries = useQueries({
    queries: productIds.map((id) => ({
      queryKey: ["product", id],
      queryFn: () => getProduct(id),
    })),
  });
}

// Dependent queries: second waits for first
function OrderDetail({ orderId }: { orderId: string }) {
  const orderQuery = useQuery({
    queryKey: ["order", orderId],
    queryFn: () => getOrder(orderId),
  });

  const userQuery = useQuery({
    queryKey: ["user", orderQuery.data?.customerId],
    queryFn: () => getUser(orderQuery.data!.customerId),
    enabled: !!orderQuery.data?.customerId, // wait for order to load
  });
}
```

### Infinite Queries for Pagination

```typescript
function InfiniteProductList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['products', 'infinite'],
    queryFn:  ({ pageParam = 1 }) =>
      productsApi.list({ page: pageParam, pageSize: 20 }),

    // Tell TQ what the next page number is
    getNextPageParam: (lastPage, pages) =>
      lastPage.meta.hasNextPage ? pages.length + 1 : undefined,

    initialPageParam: 1,
  });

  const allProducts = data?.pages.flatMap(p => p.items) ?? [];

  return (
    <>
      {allProducts.map(p => <ProductCard key={p.id} product={p} />)}
      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </>
  );
}
```

---

## 3. Query Key Design for Cache Control

Query keys are the address of your cache entries. Design them with invalidation in mind.

### Hierarchical Key Design

```typescript
// Query key factories — consistent, type-safe
export const productKeys = {
  all: () => ["products"] as const,
  lists: () => ["products", "list"] as const,
  list: (f: ProductFilters) => ["products", "list", f] as const,
  details: () => ["products", "detail"] as const,
  detail: (id: string) => ["products", "detail", id] as const,
  detailReview: (id: string) => ["products", "detail", id, "reviews"] as const,
};

export const orderKeys = {
  all: () => ["orders"] as const,
  lists: () => ["orders", "list"] as const,
  list: (f: OrderFilters) => ["orders", "list", f] as const,
  detail: (id: string) => ["orders", "detail", id] as const,
};

// Invalidation: hierarchical matching
// Invalidate specific product:
queryClient.invalidateQueries({ queryKey: productKeys.detail("prod-42") });

// Invalidate all product lists (e.g., after adding a new product):
queryClient.invalidateQueries({ queryKey: productKeys.lists() });

// Invalidate everything product-related:
queryClient.invalidateQueries({ queryKey: productKeys.all() });

// ✅ Hierarchy: invalidating a parent invalidates all children
// productKeys.all() → invalidates: lists, list(f), details, detail(id), detailReview(id)
// productKeys.lists() → invalidates: list(f) but NOT detail(id)
```

### Cache Key Serialization

```typescript
// TanStack Query serializes keys to JSON for comparison
// Keys must be stable — same data = same serialized JSON

// ❌ Unstable key: object created in render (new reference every render)
useQuery({ queryKey: ["products", { page, filters }] }); // object recreated per render
// TQ compares by JSON value, so this IS stable — but be careful with:

// ❌ Functions in keys — not serializable
useQuery({ queryKey: ["products", () => getFilter()] }); // function in key → bug

// ❌ Dates in keys: serialize to string explicitly
useQuery({ queryKey: ["events", new Date()] }); // serializes to ISO string
// Better: useQuery({ queryKey: ['events', dateToString(date)] })

// ✅ Primitives and plain objects only
useQuery({ queryKey: ["products", page, category, sortBy] });
```

---

## 4. Cache Invalidation Patterns

### Pattern 1 — Invalidate on Mutation

```typescript
// Standard pattern: mutate → invalidate related queries
const createProduct = useMutation({
  mutationFn: (data: CreateProductDto) => productsApi.create(data),

  onSuccess: (newProduct) => {
    // Invalidate all product lists — they need to show the new item
    queryClient.invalidateQueries({ queryKey: productKeys.lists() });

    // Optionally: seed the detail query immediately (no extra request needed)
    queryClient.setQueryData(productKeys.detail(newProduct.id), newProduct);
  },
});

const updateProduct = useMutation({
  mutationFn: ({ id, data }: { id: string; data: Partial<Product> }) =>
    productsApi.update(id, data),

  onSuccess: (updated) => {
    // Update specific detail (we have the data — no refetch needed)
    queryClient.setQueryData(productKeys.detail(updated.id), updated);

    // Invalidate lists — they show summary data that may have changed
    queryClient.invalidateQueries({ queryKey: productKeys.lists() });
  },
});

const deleteProduct = useMutation({
  mutationFn: (id: string) => productsApi.delete(id),

  onSuccess: (_data, deletedId) => {
    // Remove from cache (it no longer exists)
    queryClient.removeQueries({ queryKey: productKeys.detail(deletedId) });

    // Invalidate lists
    queryClient.invalidateQueries({ queryKey: productKeys.lists() });
  },
});
```

### Pattern 2 — Direct Cache Update (Avoid Refetch)

```typescript
// When you have the complete updated data from the mutation response:
// Update cache directly instead of refetching
const updateTodoMutation = useMutation({
  mutationFn: (todo: Todo) => todosApi.update(todo),

  onSuccess: (updatedTodo) => {
    // Update the specific todo in the detail cache
    queryClient.setQueryData(todoKeys.detail(updatedTodo.id), updatedTodo);

    // Update the todo within list caches
    queryClient.setQueriesData<Todo[]>(
      { queryKey: todoKeys.lists() },
      (old = []) => old.map((t) => (t.id === updatedTodo.id ? updatedTodo : t)),
    );
    // Result: zero extra network requests — cache updated from mutation response
  },
});
```

### Pattern 3 — Programmatic Refetch

```typescript
// Force-refetch specific queries
await queryClient.refetchQueries({ queryKey: productKeys.all() });

// Refetch only active (currently rendered) queries
await queryClient.refetchQueries({
  queryKey: productKeys.all(),
  type: "active",
});

// Refetch in the background (non-blocking)
queryClient.refetchQueries({ queryKey: productKeys.lists() }); // no await
```

---

## 5. Optimistic Mutations with Rollback

Show the result immediately, confirm with server, rollback on error:

```typescript
const addComment = useMutation({
  mutationFn: (comment: NewComment) => commentsApi.create(comment),

  onMutate: async (newComment) => {
    // 1. Cancel any outgoing refetches for comments (avoid race conditions)
    await queryClient.cancelQueries({
      queryKey: commentKeys.list(newComment.postId),
    });

    // 2. Snapshot the previous value (for rollback)
    const previousComments = queryClient.getQueryData<Comment[]>(
      commentKeys.list(newComment.postId),
    );

    // 3. Optimistically add the new comment
    const tempId = `temp-${Date.now()}`;
    queryClient.setQueryData<Comment[]>(
      commentKeys.list(newComment.postId),
      (old = []) => [
        ...old,
        {
          ...newComment,
          id: tempId,
          createdAt: new Date().toISOString(),
          author: currentUser, // from auth state
          isPending: true, // flag for pending UI style
        },
      ],
    );

    // 4. Return context for potential rollback
    return { previousComments, tempId };
  },

  onError: (err, newComment, context) => {
    // Rollback: restore previous state
    if (context?.previousComments) {
      queryClient.setQueryData(
        commentKeys.list(newComment.postId),
        context.previousComments,
      );
    }
    toast.error("Failed to add comment. Please try again.");
  },

  onSuccess: (createdComment, newComment, context) => {
    // Replace temp comment with real one from server
    queryClient.setQueryData<Comment[]>(
      commentKeys.list(newComment.postId),
      (old = []) =>
        old.map((c) => (c.id === context?.tempId ? createdComment : c)),
    );
  },

  onSettled: (_data, _err, newComment) => {
    // Always ensure eventually consistent
    queryClient.invalidateQueries({
      queryKey: commentKeys.list(newComment.postId),
    });
  },
});
```

---

## 6. IndexedDB for Persistent Data Cache

For offline-capable apps, persist the data cache to IndexedDB so it survives page refreshes:

```typescript
// TanStack Query + idb-keyval for persistence
import { QueryClient } from '@tanstack/react-query';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';
import { PersistQueryClientProvider, persistQueryClient } from '@tanstack/react-query-persist-client';
import { get, set, del, createStore } from 'idb-keyval';

// Custom IndexedDB persister (more reliable than localStorage for large data)
const idbStore = createStore('rq-cache', 'tanstack-query');

const idbPersister = {
  persistClient: async (client: string) => set('client', client, idbStore),
  restoreClient: async () => get<string>('client', idbStore),
  removeClient:  async () => del('client', idbStore),
};

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 60 * 24, // 24 hours — longer since we persist
    },
  },
});

// App root:
function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister:          idbPersister,
        maxAge:             1000 * 60 * 60 * 24, // 24 hours max age
        buster:             APP_VERSION,          // clear on app update
        dehydrateOptions: {
          // Only persist important queries
          shouldDehydrateQuery: (query) =>
            query.state.status === 'success' &&
            query.queryKey[0] !== 'user', // don't persist user-specific data
        },
      }}
    >
      <AppContent />
    </PersistQueryClientProvider>
  );
}
```

### Direct IndexedDB Usage

```typescript
// For custom persistence needs beyond TanStack Query
import { openDB, IDBPDatabase } from "idb";

class DataCache {
  #db: IDBPDatabase | null = null;
  #dbName = "app-data-cache";
  #version = 1;

  async #getDB(): Promise<IDBPDatabase> {
    if (!this.#db) {
      this.#db = await openDB(this.#dbName, this.#version, {
        upgrade(db) {
          // Create object stores
          const cacheStore = db.createObjectStore("cache", { keyPath: "key" });
          cacheStore.createIndex("expiresAt", "expiresAt");
        },
      });
    }
    return this.#db;
  }

  async set<T>(key: string, data: T, ttlMs = 5 * 60_000): Promise<void> {
    const db = await this.#getDB();
    await db.put("cache", { key, data, expiresAt: Date.now() + ttlMs });
  }

  async get<T>(key: string): Promise<T | null> {
    const db = await this.#getDB();
    const entry = await db.get("cache", key);
    if (!entry) return null;
    if (Date.now() > entry.expiresAt) {
      await db.delete("cache", key);
      return null;
    }
    return entry.data as T;
  }

  async purgeExpired(): Promise<number> {
    const db = await this.#getDB();
    const expired = await db.getAllFromIndex(
      "cache",
      "expiresAt",
      IDBKeyRange.upperBound(Date.now()),
    );
    const tx = db.transaction("cache", "readwrite");
    await Promise.all(expired.map((e) => tx.store.delete(e.key)));
    await tx.done;
    return expired.length;
  }

  async clear(): Promise<void> {
    const db = await this.#getDB();
    await db.clear("cache");
  }
}

export const dataCache = new DataCache();

// Usage
const products = await dataCache.get<Product[]>("products:electronics");
if (!products) {
  const fresh = await productsApi.list({ category: "electronics" });
  await dataCache.set("products:electronics", fresh, 5 * 60_000);
  return fresh;
}
return products;
```

---

## 7. Cache Synchronization Across Tabs

Multiple browser tabs need to stay in sync when data changes:

```typescript
// BroadcastChannel: communicate between tabs on the same origin
class CrossTabCacheSync {
  #channel = new BroadcastChannel("cache-sync");
  #queryClient: QueryClient;

  constructor(queryClient: QueryClient) {
    this.#queryClient = queryClient;
    this.#channel.onmessage = this.#handleMessage.bind(this);
  }

  // Broadcast an invalidation to all other tabs
  broadcast(event: CacheSyncEvent): void {
    this.#channel.postMessage(event);
  }

  #handleMessage(event: MessageEvent<CacheSyncEvent>): void {
    const { type, queryKey } = event.data;

    switch (type) {
      case "INVALIDATE":
        this.#queryClient.invalidateQueries({ queryKey });
        break;
      case "REMOVE":
        this.#queryClient.removeQueries({ queryKey });
        break;
      case "SET_DATA":
        this.#queryClient.setQueryData(queryKey, event.data.data);
        break;
    }
  }

  destroy(): void {
    this.#channel.close();
  }
}

type CacheSyncEvent =
  | { type: "INVALIDATE"; queryKey: unknown[] }
  | { type: "REMOVE"; queryKey: unknown[] }
  | { type: "SET_DATA"; queryKey: unknown[]; data: unknown };

// Initialize with the query client
const cacheSync = new CrossTabCacheSync(queryClient);

// After mutation: broadcast to other tabs
const updateProduct = useMutation({
  mutationFn: (product: Product) => productsApi.update(product.id, product),

  onSuccess: (updated) => {
    queryClient.setQueryData(productKeys.detail(updated.id), updated);
    queryClient.invalidateQueries({ queryKey: productKeys.lists() });

    // Sync to other tabs
    cacheSync.broadcast({
      type: "SET_DATA",
      queryKey: productKeys.detail(updated.id),
      data: updated,
    });
    cacheSync.broadcast({ type: "INVALIDATE", queryKey: productKeys.lists() });
  },
});
```

---

## 8. Offline-First Data Architecture

```typescript
// Offline-first: assume the network might not work
// Strategy: read from cache first, write to queue if offline

class OfflineFirstDataManager {
  #queue: MutationQueueItem[] = [];
  #isOnline: boolean = navigator.onLine;

  constructor(private queryClient: QueryClient) {
    window.addEventListener("online", () => {
      this.#isOnline = true;
      this.#processQueue();
    });
    window.addEventListener("offline", () => {
      this.#isOnline = false;
    });
  }

  async mutate<T>(
    key: string,
    mutateFn: () => Promise<T>,
    optimisticUpdate: () => void,
    rollback: () => void,
  ): Promise<T | null> {
    // Apply optimistic update immediately
    optimisticUpdate();

    if (this.#isOnline) {
      try {
        return await mutateFn();
      } catch (err) {
        rollback();
        throw err;
      }
    } else {
      // Offline: queue for later
      this.#queue.push({ key, mutateFn, rollback, id: Date.now().toString() });
      await this.#persistQueue();
      return null; // will be confirmed later
    }
  }

  async #processQueue(): Promise<void> {
    while (this.#queue.length > 0 && this.#isOnline) {
      const item = this.#queue.shift()!;
      try {
        await item.mutateFn();
      } catch (err) {
        item.rollback();
        console.error(`Queue item ${item.key} failed:`, err);
      }
    }
    await this.#persistQueue();
  }

  async #persistQueue(): Promise<void> {
    // Persist queue to IndexedDB for survival across refreshes
    await dataCache.set("mutation-queue", this.#queue, 24 * 60 * 60_000);
  }

  async restoreQueue(): Promise<void> {
    const saved = await dataCache.get<MutationQueueItem[]>("mutation-queue");
    if (saved?.length) this.#queue = saved;
  }
}

interface MutationQueueItem {
  id: string;
  key: string;
  mutateFn: () => Promise<unknown>;
  rollback: () => void;
}
```

---

## 9. Real-Time Cache Updates via WebSocket

```typescript
// WebSocket events → TanStack Query cache updates
class RealtimeCacheUpdater {
  constructor(
    private queryClient: QueryClient,
    private wsUrl: string,
  ) {}

  connect(authToken: string): void {
    const ws = new WebSocket(`${this.wsUrl}?token=${authToken}`);

    ws.onmessage = ({ data }) => {
      const event: ServerPushEvent = JSON.parse(data);
      this.#handleEvent(event);
    };
  }

  #handleEvent(event: ServerPushEvent): void {
    switch (event.type) {
      case "PRODUCT_UPDATED":
        // Update cache with pushed data (no refetch needed)
        this.queryClient.setQueryData(
          productKeys.detail(event.payload.id),
          event.payload,
        );
        // Invalidate lists (summary data may have changed)
        this.queryClient.invalidateQueries({ queryKey: productKeys.lists() });
        break;

      case "ORDER_STATUS_CHANGED":
        this.queryClient.setQueryData(
          orderKeys.detail(event.payload.orderId),
          (old: Order | undefined) =>
            old ? { ...old, status: event.payload.newStatus } : undefined,
        );
        break;

      case "NEW_NOTIFICATION":
        // Prepend to notifications list
        this.queryClient.setQueryData<Notification[]>(
          ["notifications"],
          (old = []) => [event.payload, ...old].slice(0, 100), // max 100
        );
        // Update unread count
        this.queryClient.setQueryData<number>(
          ["notifications", "unread-count"],
          (old = 0) => old + 1,
        );
        break;
    }
  }
}

type ServerPushEvent =
  | { type: "PRODUCT_UPDATED"; payload: Product }
  | {
      type: "ORDER_STATUS_CHANGED";
      payload: { orderId: string; newStatus: OrderStatus };
    }
  | { type: "NEW_NOTIFICATION"; payload: Notification };
```

---

## 10. Cache Warming Strategies

```typescript
// 1. Prefetch on app load (critical data)
function AppProviders({ children }: { children: ReactNode }) {
  const queryClient = useQueryClient();

  useEffect(() => {
    // Warm critical caches immediately on app load
    queryClient.prefetchQuery({
      queryKey: ['navigation'],
      queryFn:  () => contentApi.getNavigation(),
      staleTime: 60_000,
    });
    queryClient.prefetchQuery({
      queryKey: ['user-permissions'],
      queryFn:  () => authApi.getPermissions(),
      staleTime: 5 * 60_000,
    });
  }, [queryClient]);

  return <>{children}</>;
}

// 2. Prefetch on hover (next likely page)
function CategoryLink({ category }: { category: Category }) {
  const queryClient = useQueryClient();

  const prefetch = useCallback(() => {
    queryClient.prefetchQuery({
      queryKey:  productKeys.list({ category: category.id }),
      queryFn:   () => productsApi.list({ category: category.id }),
      staleTime: 30_000,
    });
  }, [category.id, queryClient]);

  return (
    <Link href={`/category/${category.id}`} onMouseEnter={prefetch}>
      {category.name}
    </Link>
  );
}

// 3. Prefetch adjacent pages during idle time
function usePaginationPrefetch(currentPage: number, totalPages: number) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const nextPage = currentPage + 1;
    const prevPage = currentPage - 1;

    requestIdleCallback(() => {
      if (nextPage <= totalPages) {
        queryClient.prefetchQuery({
          queryKey: productKeys.list({ page: nextPage }),
          queryFn:  () => productsApi.list({ page: nextPage }),
        });
      }
      if (prevPage >= 1) {
        queryClient.prefetchQuery({
          queryKey: productKeys.list({ page: prevPage }),
          queryFn:  () => productsApi.list({ page: prevPage }),
        });
      }
    });
  }, [currentPage, totalPages, queryClient]);
}
```

---

## 11. Cache Debugging and Inspection

```typescript
// React Query DevTools in development
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AppContent />
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} buttonPosition="bottom-right" />
      )}
    </QueryClientProvider>
  );
}

// Programmatic cache inspection
function debugCache() {
  const cache = queryClient.getQueryCache();

  console.group('Query Cache State');
  cache.getAll().forEach(query => {
    const { queryKey, state } = query;
    console.log({
      key:        JSON.stringify(queryKey),
      status:     state.status,
      dataAge:    state.dataUpdatedAt
        ? `${((Date.now() - state.dataUpdatedAt) / 1000).toFixed(0)}s ago`
        : 'never',
      isStale:    query.isStale(),
      observers:  query.getObserversCount(),
    });
  });
  console.groupEnd();
}

// Cache hit/miss logging for profiling
const originalSetQueryData = queryClient.setQueryData.bind(queryClient);
queryClient.setQueryData = function(...args) {
  console.debug('[Cache SET]', args[0]);
  return originalSetQueryData(...args);
};
```

---

## 12. Good Practices

### ✅ Set appropriate staleTime per data type

```typescript
// Data that changes rarely → longer staleTime
useQuery({ staleTime: 60 * 60_000 }); // 1 hour for reference data

// Data that changes frequently → shorter staleTime
useQuery({ staleTime: 30_000 }); // 30 seconds for live data

// Data that must always be fresh → staleTime: 0 (default)
useQuery({ staleTime: 0 }); // always consider stale, always refetch on mount
```

### ✅ Use `select` for cache access pattern optimization

```typescript
// Components subscribe to only the slice they need
// Re-renders only when that slice changes

const productName = useQuery({
  queryKey: productKeys.detail(productId),
  queryFn: () => productsApi.get(productId),
  select: (data) => data.name, // only name — re-renders on name change only
});
```

### ✅ Seed the detail cache from list data

```typescript
// After loading a list: pre-populate individual detail queries
const { data: products } = useQuery({
  queryKey: productKeys.lists(),
  queryFn: () => productsApi.list(),
  onSuccess: (products) => {
    // Seed individual detail caches from list data
    products.forEach((product) => {
      queryClient.setQueryData(productKeys.detail(product.id), product, {
        updatedAt: Date.now(),
      });
    });
  },
});
// Now navigating to a product detail: instant cache hit
```

---

## 13. Bad Practices

### ❌ Using query cache for local UI state

```typescript
// ❌ Query cache for UI state — wrong tool
queryClient.setQueryData(["sidebar-open"], true);
queryClient.setQueryData(["selected-tab"], "reviews");
// Query cache is for server state — these are UI state (useState/Zustand)
```

### ❌ Invalidating too broadly

```typescript
// ❌ Invalidates ALL queries after any mutation — forces all data to refetch
queryClient.invalidateQueries(); // no filter!

// ✅ Invalidate only affected queries
queryClient.invalidateQueries({ queryKey: productKeys.lists() });
queryClient.setQueryData(productKeys.detail(id), updatedData);
```

### ❌ Not handling stale cache on user switching

```typescript
// ❌ User A's cache shown to User B after switching accounts
// If User B logs in and user changes without clearing cache

// ✅ Clear all user-specific cache on logout/user switch
function onUserChange() {
  queryClient.clear(); // removes all cached data
  // Or: selectively clear user-specific keys:
  queryClient.removeQueries({ queryKey: ["user"] });
  queryClient.removeQueries({ queryKey: ["orders"] });
}
```

---

## 14. Common Mistakes

### Mistake 1 — Not cancelling in-flight queries during optimistic updates

```typescript
// ❌ Optimistic update without cancelling in-flight queries
onMutate: async (newData) => {
  // If a refetch was in progress: it may overwrite our optimistic update!
  queryClient.setQueryData(key, newData); // race condition risk

  // ✅ Cancel any in-flight queries first
  await queryClient.cancelQueries({ queryKey: key });
  queryClient.setQueryData(key, newData);
},
```

### Mistake 2 — Using `initialData` when `placeholderData` is more appropriate

```typescript
// initialData: treated as real data, sets updatedAt → may not refetch
useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
  initialData: cachedProducts, // sets dataUpdatedAt → may skip initial fetch!
});

// placeholderData: shown temporarily until real data arrives → always fetches
useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
  placeholderData: cachedProducts, // shown while fetching, replaced by real data
});

// Use initialData when you KNOW the data is fresh
// Use placeholderData when you want something to show while fetching
```

### Mistake 3 — Not handling query deduplication correctly

```typescript
// TanStack Query deduplicates: 5 components requesting the same key = 1 network request
// But: keys must be IDENTICAL for deduplication to work

// ❌ Different object references: not deduplicated (different JSON)
useQuery({ queryKey: ["products", { category: "a" }] }); // component 1
useQuery({ queryKey: ["products", { category: "a" }] }); // component 2
// These ARE deduplicated — TQ compares by JSON value

// ❌ These are NOT deduplicated (different JSON):
useQuery({ queryKey: ["products", { category: "a", page: undefined }] });
useQuery({ queryKey: ["products", { category: "a" }] });
// { category: 'a', page: undefined } ≠ { category: 'a' }

// ✅ Use query key factories to ensure consistent keys
useQuery({ queryKey: productKeys.list({ category: "a" }) }); // consistent
```

---

## 15. Interview-Level Explanation

> **"How do you approach data caching in a large frontend application?"**

**Strong answer:**

> "Data caching in large frontend applications is primarily about server state management — the data that lives on the server, gets fetched asynchronously, and can become stale. The right tool for this is TanStack Query or SWR, not Redux or Zustand. These libraries give you a cache keyed by query identifiers, with configurable staleness, automatic background refresh, deduplication of concurrent requests, and mutation handling.
>
> The most important design decision is query key structure. I use hierarchical factory functions: `productKeys.detail(id)` produces `['products', 'detail', id]`. This hierarchy enables precise invalidation — invalidating `['products', 'detail']` hits all detail queries, while `['products']` hits everything. After a mutation, you invalidate exactly what changed — not more, not less.
>
> For optimistic updates, the pattern is: cancel in-flight queries for the affected key, snapshot the previous data, apply the optimistic update to the cache, then on error rollback to the snapshot. On success, either update the cache directly from the server response or invalidate to trigger a fresh fetch. The key discipline is calling `cancelQueries` before the optimistic update to avoid race conditions where an in-flight refetch overwrites your optimistic state.
>
> Cache invalidation across features is handled by publishing cache update events or using hierarchical query keys. After a product update, I invalidate the product detail and product lists. Any component subscribed to those keys will automatically refetch.
>
> For offline support, the persistent cache pattern stores the TanStack Query cache in IndexedDB using the persist-client plugin. The cache survives page refreshes, giving users their last known data immediately. Failed mutations get queued and retried when connectivity is restored.
>
> The staleTime configuration is the key tuning lever. Set it to 0 and every mount triggers a refetch. Set it to 5 minutes and data is served from cache for 5 minutes before background refresh. I set it based on how often data actually changes — reference data like countries or categories gets a 24-hour staleTime; real-time data like inventory gets 30 seconds."

---

## 16. Exercises

### Exercise 1 — Design cache invalidation

A user updates their profile photo. The photo appears in:

1. The header avatar
2. The profile page
3. Comments the user has left (each shows their avatar)
4. The user's public profile page

Design the query key structure and invalidation strategy:

<details>
<summary>Solution</summary>

```typescript
// Query key factories
const userKeys = {
  all: () => ["users"] as const,
  detail: (id: string) => ["users", "detail", id] as const,
  avatar: (id: string) => ["users", "detail", id, "avatar"] as const,
  public: (username: string) => ["users", "public", username] as const,
};

const commentKeys = {
  all: () => ["comments"] as const,
  byPost: (postId: string) => ["comments", "post", postId] as const,
};

// On successful photo update:
const updateAvatar = useMutation({
  mutationFn: (file: File) => userApi.uploadAvatar(file),

  onSuccess: (updatedUser) => {
    // 1. Update the user detail cache (header reads from this)
    queryClient.setQueryData(userKeys.detail(currentUser.id), (old: User) => ({
      ...old,
      avatarUrl: updatedUser.avatarUrl,
    }));

    // 2. Update avatar-specific cache if exists
    queryClient.setQueryData(
      userKeys.avatar(currentUser.id),
      updatedUser.avatarUrl,
    );

    // 3. Invalidate public profile page (external viewers)
    queryClient.invalidateQueries({
      queryKey: userKeys.public(currentUser.username),
    });

    // 4. For comments: two strategies:
    //    a) Invalidate all comments (heavy — refetches all pages)
    //    b) Update each comment in-cache that shows this user's avatar
    //    For a single user changing their own avatar: option b is better

    queryClient.setQueriesData<Comment[]>(
      { queryKey: commentKeys.all() },
      (comments = []) =>
        comments.map((comment) =>
          comment.authorId === currentUser.id
            ? {
                ...comment,
                author: { ...comment.author, avatarUrl: updatedUser.avatarUrl },
              }
            : comment,
        ),
    );

    toast.success("Profile photo updated!");
  },
});

// This approach:
// - Updates header avatar instantly (direct cache update)
// - Updates profile page cache (direct update)
// - Invalidates public profile (server has final version)
// - Updates all comment author avatars without refetching all comments
// - Zero unnecessary network requests
```

</details>

---

## 🔗 Related Topics

- [`caching/03-memory-caching.md`](./03-memory-caching.md) — In-memory caching for derived state
- [`caching/01-http-caching.md`](./01-http-caching.md) — HTTP layer caching
- [`system-design/04-state-management-design.md`](../system-design/04-state-management-design.md) — Server vs client state distinction
- [`caching/05-cdn-strategies.md`](./05-cdn-strategies.md) — CDN as a data cache layer
- [`javascript-core/13-service-workers.md`](../javascript-core/13-service-workers.md) — Service Workers for offline data

---

<div align="center">

**Next:** [`caching/05-cdn-strategies.md`](./05-cdn-strategies.md) →

</div>
