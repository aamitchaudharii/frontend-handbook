# 03 — System Design Interview Questions

> **"Frontend system design is not about knowing the right answer — it's about demonstrating that you can reason through tradeoffs. The interviewer wants to see how you navigate ambiguity, ask clarifying questions, consider competing constraints, and justify decisions. A mediocre design with excellent reasoning beats a perfect design with no explanation."**

Frontend system design interviews ask you to design scalable, maintainable frontend architectures — from component hierarchies and state management to performance strategies, offline support, and deployment. This document covers the most common question types, frameworks for approaching them, and detailed example answers that demonstrate senior-level reasoning.

---

## 📚 Table of Contents

1. [The System Design Interview Framework](#1-the-system-design-interview-framework)
2. [Design a News Feed (Facebook/Twitter-style)](#2-design-a-news-feed-facebooktwitter-style)
3. [Design a Typeahead / Autocomplete](#3-design-a-typeahead--autocomplete)
4. [Design a Photo Upload Component](#4-design-a-photo-upload-component)
5. [Design a Real-Time Collaborative Editor](#5-design-a-real-time-collaborative-editor)
6. [Design a Data Table Component](#6-design-a-data-table-component)
7. [Design for Offline Support](#7-design-for-offline-support)
8. [Performance Design Questions](#8-performance-design-questions)
9. [Component API Design Questions](#9-component-api-design-questions)
10. [Architecture Tradeoff Questions](#10-architecture-tradeoff-questions)

---

## 1. The System Design Interview Framework

```
BEFORE DESIGNING:

1. CLARIFY REQUIREMENTS (5-10 minutes)
   - What is the primary use case? Who is the user?
   - What scale? (100 users? 100M users?)
   - What constraints? (accessibility? mobile-first? offline support?)
   - What is the team context? (new codebase? existing design system?)

2. DEFINE SUCCESS CRITERIA
   - Core functional requirements (must have)
   - Non-functional requirements (performance, reliability, accessibility)
   - Out of scope (what you're explicitly NOT designing)

3. STRUCTURE YOUR RESPONSE
   Component Architecture → Data Flow → State Management → API Design →
   Performance Considerations → Error Handling → Accessibility →
   Future Considerations

DURING THE DESIGN:
  - Narrate your thinking: "I'm choosing X over Y because..."
  - Acknowledge tradeoffs: "This approach has the downside of..."
  - Ask clarifying questions when truly ambiguous (not to stall)
  - Show iterative thinking: "Initially I'd do X, then optimize to Y"
```

---

## 2. Design a News Feed (Facebook/Twitter-style)

**Interviewer:** "Design the frontend for a social media news feed."

### Clarifying Questions

```
- How many posts are expected per load? (~20, with infinite scroll)
- What types of content? (text, images, videos, polls, shares)
- Real-time updates? (new posts appear without refresh)
- Offline support needed?
- Mobile app or web?
```

### Architecture

```
COMPONENT HIERARCHY:
  <FeedPage>
    <FeedFilters />           (tab navigation: Following, Trending, etc.)
    <FeedComposer />          (new post input)
    <VirtualizedFeedList>     (virtualized for performance)
      <FeedItem>              (each post)
        <PostHeader />        (avatar, name, timestamp)
        <PostContent />       (text, images, video)
        <PostActions />       (like, comment, share — interactions)
        <CommentsPreview />   (3-5 comments, expandable)
      </FeedItem>
    </VirtualizedFeedList>
    <NewPostsNotification />  (real-time: "15 new posts — click to refresh")
  </FeedPage>
```

### Data Fetching & State

```typescript
// Infinite scroll with TanStack Query
function useFeed(filter: FeedFilter) {
  return useInfiniteQuery({
    queryKey: ["feed", filter],
    queryFn: ({ pageParam = null }) =>
      feedApi.list({ filter, cursor: pageParam, limit: 20 }),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 30_000, // 30s: don't re-fetch on every focus
    refetchOnWindowFocus: false,
  });
}

// Real-time new posts (SSE or polling)
function useFeedUpdates(filter: FeedFilter, onNew: (count: number) => void) {
  useEffect(() => {
    const source = new EventSource(`/api/feed/updates?filter=${filter}`);
    source.onmessage = (e) => {
      const { newPostCount } = JSON.parse(e.data);
      if (newPostCount > 0) onNew(newPostCount);
    };
    return () => source.close();
  }, [filter]);
}
```

### Performance Design

```
VIRTUALIZATION: render only visible posts
  - Use react-window or @tanstack/virtual
  - Each post: variable height — use dynamic measurement
  - Keep ~5 posts off-screen in each direction (overscan)
  - At 1000 posts: render only ~15 DOM nodes

IMAGE LOADING:
  - Lazy load images below fold (loading="lazy")
  - Low-quality placeholder → blur up (LQIP pattern)
  - Next-gen formats: WebP, AVIF via CDN transformation

OPTIMISTIC UPDATES:
  - Like/unlike: update UI immediately, confirm with server
  - Rollback if server rejects

FEED FRESHNESS:
  - Don't inject new posts into the visible list (jarring UX)
  - Show "15 new posts" banner → user clicks → prepend + scroll to top
  - Prevent CLS (Cumulative Layout Shift) from injected content
```

---

## 3. Design a Typeahead / Autocomplete

**Interviewer:** "Design a typeahead/autocomplete search component."

### Architecture

```typescript
// Core concerns:
// 1. Debouncing (don't fetch on every keystroke)
// 2. Request cancellation (avoid stale results)
// 3. Keyboard navigation
// 4. Accessibility (ARIA)
// 5. Caching (avoid redundant requests)

interface AutocompleteProps {
  onSelect:    (value: Option) => void;
  fetchOptions: (query: string) => Promise<Option[]>;
  debounceMs?:  number; // default 300
  minChars?:    number; // default 2
  maxResults?:  number; // default 10
  placeholder?: string;
}

function Autocomplete({
  onSelect, fetchOptions, debounceMs = 300, minChars = 2, maxResults = 10
}: AutocompleteProps) {
  const [query, setQuery]         = useState('');
  const [options, setOptions]     = useState<Option[]>([]);
  const [isOpen, setIsOpen]       = useState(false);
  const [activeIndex, setIndex]   = useState(-1);
  const [isLoading, setLoading]   = useState(false);
  const cache                     = useRef<Map<string, Option[]>>(new Map());
  const abortRef                  = useRef<AbortController | null>(null);

  // Debounced fetch
  useEffect(() => {
    if (query.length < minChars) { setOptions([]); setIsOpen(false); return; }

    const timer = setTimeout(async () => {
      // Check cache first
      if (cache.current.has(query)) {
        setOptions(cache.current.get(query)!.slice(0, maxResults));
        setIsOpen(true);
        return;
      }

      // Cancel previous in-flight request
      abortRef.current?.abort();
      abortRef.current = new AbortController();

      setLoading(true);
      try {
        const results = await fetchOptions(query);
        cache.current.set(query, results);
        setOptions(results.slice(0, maxResults));
        setIsOpen(true);
      } catch (err) {
        if (err.name !== 'AbortError') setOptions([]);
      } finally {
        setLoading(false);
      }
    }, debounceMs);

    return () => { clearTimeout(timer); abortRef.current?.abort(); };
  }, [query, debounceMs, minChars, maxResults, fetchOptions]);

  // Keyboard navigation
  function handleKeyDown(e: React.KeyboardEvent) {
    if (!isOpen) return;
    if (e.key === 'ArrowDown') { e.preventDefault(); setIndex(i => Math.min(i + 1, options.length - 1)); }
    if (e.key === 'ArrowUp')   { e.preventDefault(); setIndex(i => Math.max(i - 1, -1)); }
    if (e.key === 'Enter' && activeIndex >= 0) { onSelect(options[activeIndex]); setIsOpen(false); }
    if (e.key === 'Escape')    { setIsOpen(false); }
  }

  return (
    <div role="combobox" aria-expanded={isOpen} aria-haspopup="listbox">
      <input
        value={query}
        onChange={e => { setQuery(e.target.value); setIndex(-1); }}
        onKeyDown={handleKeyDown}
        aria-autocomplete="list"
        aria-controls="autocomplete-list"
        aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
      />
      {isLoading && <Spinner />}
      {isOpen && (
        <ul id="autocomplete-list" role="listbox">
          {options.map((opt, i) => (
            <li
              key={opt.id}
              id={`option-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              onClick={() => { onSelect(opt); setIsOpen(false); }}
            >
              {opt.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## 4. Design a Photo Upload Component

**Interviewer:** "Design a component for uploading multiple photos."

### Architecture Considerations

```
REQUIREMENTS TO CLARIFY:
  Max file count? (10 photos)
  Max file size? (10MB each)
  Accepted formats? (JPEG, PNG, WEBP)
  Progress tracking? (yes, per file)
  Preview before upload? (yes)
  Drag and drop?
  Cancel individual uploads?

KEY DESIGN DECISIONS:

1. VALIDATION (client-side first):
   - File type (MIME type, not just extension)
   - File size
   - Dimensions (if image-specific constraints)
   - Total count

2. UPLOAD STRATEGY:
   - Parallel uploads (all at once): faster but higher server load
   - Sequential: slower but controlled
   - Chunked uploads (large files): resume on network drop

3. PROGRESS TRACKING:
   - XHR provides upload progress events
   - Fetch API: limited support for upload progress
   - Use XHR for progress, wrap in a Promise

4. UI STATES PER FILE:
   Pending → Uploading (% progress) → Success → Error → Cancelled

5. PREVIEW:
   - FileReader.readAsDataURL or URL.createObjectURL (faster, revoke when done)
   - Revoke object URLs when component unmounts (memory management)
```

### Implementation Sketch

```typescript
interface UploadFile {
  id:        string;
  file:      File;
  preview:   string;         // object URL
  status:    'pending' | 'uploading' | 'done' | 'error' | 'cancelled';
  progress:  number;         // 0-100
  error?:    string;
}

function PhotoUploader({ onUploadComplete, maxFiles = 10 }) {
  const [files, setFiles] = useState<UploadFile[]>([]);
  const abortControllers = useRef<Map<string, XMLHttpRequest>>(new Map());

  function addFiles(newFiles: FileList) {
    const remaining = maxFiles - files.length;
    const valid = Array.from(newFiles).slice(0, remaining).filter(validateFile);

    const uploadFiles = valid.map(file => ({
      id:       crypto.randomUUID(),
      file,
      preview:  URL.createObjectURL(file), // fast, memory-efficient
      status:   'pending' as const,
      progress: 0,
    }));

    setFiles(prev => [...prev, ...uploadFiles]);
    uploadFiles.forEach(uploadFile); // start uploads
  }

  function uploadFile(uploadFile: UploadFile) {
    const xhr = new XMLHttpRequest();
    abortControllers.current.set(uploadFile.id, xhr);

    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        setFiles(prev => prev.map(f =>
          f.id === uploadFile.id ? { ...f, progress: Math.round(e.loaded / e.total * 100), status: 'uploading' } : f
        ));
      }
    };

    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        const { url } = JSON.parse(xhr.responseText);
        setFiles(prev => prev.map(f =>
          f.id === uploadFile.id ? { ...f, status: 'done', progress: 100 } : f
        ));
        onUploadComplete(uploadFile.id, url);
      } else {
        setFiles(prev => prev.map(f =>
          f.id === uploadFile.id ? { ...f, status: 'error', error: 'Upload failed' } : f
        ));
      }
    };

    const formData = new FormData();
    formData.append('file', uploadFile.file);
    xhr.open('POST', '/api/photos/upload');
    xhr.send(formData);
  }

  function cancelUpload(id: string) {
    abortControllers.current.get(id)?.abort();
    setFiles(prev => prev.map(f => f.id === id ? { ...f, status: 'cancelled' } : f));
  }

  // Cleanup object URLs
  useEffect(() => () => files.forEach(f => URL.revokeObjectURL(f.preview)), []);

  return (/* render upload UI */);
}
```

---

## 5. Design a Real-Time Collaborative Editor

**Interviewer:** "How would you design a collaborative text editor (like Google Docs)?"

### Key Challenges

```
1. CONFLICT RESOLUTION: two users edit simultaneously
   → Operational Transformation (OT) or CRDTs (Conflict-free Replicated Data Types)
   → OT: server reconciles operations, reorders/transforms them
   → CRDTs: no central coordination needed, merge automatically

2. PRESENCE: show which users are online and where they're editing
   → WebSocket for real-time presence updates
   → Cursor positions broadcast to all participants

3. OFFLINE EDITING: user goes offline, continues editing, reconnects
   → Buffer changes locally
   → Apply on reconnect with appropriate conflict resolution

4. LATENCY HIDING: changes should appear immediately (optimistic)
   → Apply own changes locally before server confirms
   → OT/CRDT ensures eventual consistency
```

### Architecture

```
FRONTEND ARCHITECTURE:

Editor Core:
  - Lexical / Slate.js / ProseMirror as the editor primitive
  - Our sync layer on top, independent of editor choice

State layers:
  - Local editor state: what you've typed (not yet confirmed)
  - Committed state: operations confirmed by server
  - Pending operations: operations sent to server, not yet confirmed

WEBSOCKET PROTOCOL:
  Client → Server: { type: 'operation', op: {...}, revision: 42 }
  Server → Client: { type: 'operation', op: {...}, revision: 43 } (from another user)
  Server → Client: { type: 'ack', revision: 43 } (confirms my operation was accepted)

OPERATIONAL TRANSFORMATION (simplified):
  User A: insert 'X' at position 5 (at revision 10)
  User B: delete char at position 3 (at revision 10) — sent to server simultaneously

  Server receives A's op first: commits at revision 11
  Server receives B's op: must TRANSFORM B's delete relative to A's insert
  A's insert shifted all positions after 5 by +1
  B's delete was at position 3 — not affected (3 < 5) → unchanged
  Server commits transformed B's op at revision 12
  Server sends A's transformed B op so A can apply it locally

PRESENCE:
  WebSocket event: { type: 'cursor', userId: 'u2', position: 42 }
  Show other users' cursors as colored carets with their avatar/name
  Throttle cursor broadcasts: max 10/second per user
```

---

## 6. Design a Data Table Component

**Interviewer:** "Design a reusable, performant data table component."

### API Design

```typescript
// GOAL: flexible enough for many use cases, opinionated enough to be useful

interface DataTableProps<T> {
  // Data
  data:         T[];
  columns:      ColumnDef<T>[];

  // Performance (large datasets)
  virtualized?: boolean;    // enable virtual scrolling
  rowHeight?:   number;     // for fixed-height virtualization

  // Sorting
  sortable?:    boolean;
  defaultSort?: { field: keyof T; direction: 'asc' | 'desc' };
  onSort?:      (sort: SortConfig<T>) => void; // for server-side sort

  // Filtering
  filterable?:  boolean;
  filterState?: FilterState<T>; // controlled
  onFilter?:    (filters: FilterState<T>) => void;

  // Pagination
  pagination?:  PaginationConfig; // { page, pageSize, total, onChange }

  // Row features
  selectable?:  boolean;
  onSelect?:    (rows: T[]) => void;
  expandable?:  (row: T) => ReactNode; // render expanded content

  // Customization
  renderEmpty?: ReactNode;
  loading?:     boolean;
  className?:   string;
}

interface ColumnDef<T> {
  key:        keyof T | string;
  header:     ReactNode;
  cell?:      (value: unknown, row: T) => ReactNode; // custom renderer
  sortable?:  boolean;
  width?:     number | string;
  align?:     'left' | 'center' | 'right';
  sticky?:    'left' | 'right';     // sticky columns
  pinned?:    boolean;
}

// Usage:
<DataTable
  data={users}
  columns={[
    { key: 'name', header: 'Name', sortable: true },
    { key: 'email', header: 'Email' },
    { key: 'role', header: 'Role',
      cell: (value) => <RoleBadge role={value as string} /> },
    { key: 'actions', header: '',
      cell: (_, user) => <UserActions user={user} /> },
  ]}
  virtualized
  selectable
  onSelect={handleSelect}
  pagination={{ page: 1, pageSize: 50, total: 10000, onChange: setPage }}
/>
```

### Performance Strategy

```
FOR 10,000+ ROWS:

1. VIRTUALIZATION: only render visible rows
   @tanstack/virtual: flexible, supports variable height
   react-window: simpler, fixed height only

2. COLUMN VIRTUALIZATION (very wide tables):
   Similar to row virtualization but horizontal

3. MEMOIZATION:
   Each row wrapped in React.memo
   Cell renderers stable via useCallback or static functions

4. FILTERING/SORTING:
   Client-side: useMemo to avoid recalculation on every render
   Server-side: controlled filter/sort state, debounce changes before fetching

5. COLUMN RESIZING:
   Track column widths in state
   Use CSS variables for efficient updates (no JS per-cell)
```

---

## 7. Design for Offline Support

**Interviewer:** "How would you add offline support to a web application?"

### Layered Strategy

```
LAYER 1: SERVICE WORKER (static assets)
  Cache: JS bundles, CSS, fonts, images
  Strategy: Cache-first with content hashing
  Result: app shell loads instantly, even offline

LAYER 2: DATA CACHING (TanStack Query + IndexedDB)
  Persist the query cache to IndexedDB
  On load: restore last known data immediately
  Background: refresh when online
  Result: users see last-known data when offline

LAYER 3: OPTIMISTIC MUTATIONS with offline queue
  User actions (create, update, delete): apply optimistically
  If offline: queue the mutation for later
  On reconnect: drain the queue in order
  On conflict: show resolution UI

LAYER 4: NETWORK STATUS AWARENESS
  navigator.onLine + online/offline events
  Show banner: "You're offline — changes will sync when you reconnect"
  Disable features that require real-time data (live chat, prices)
```

### Implementation

```typescript
// Detect online status
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  useEffect(() => {
    window.addEventListener("online", () => setIsOnline(true));
    window.addEventListener("offline", () => setIsOnline(false));
  }, []);
  return isOnline;
}

// Offline mutation queue
async function performMutation(mutationFn, payload) {
  if (navigator.onLine) {
    return mutationFn(payload); // online: do immediately
  }
  // Offline: queue for later
  await offlineQueue.add({ mutationFn: mutationFn.name, payload });
  await registration.sync.register("sync-mutations"); // Background Sync API
  return { queued: true }; // return immediately, will sync later
}
```

---

## 8. Performance Design Questions

### Q: "How would you optimize a page with a 3-second LCP?"

```
SYSTEMATIC APPROACH:

1. DIAGNOSE first (don't guess)
   Chrome DevTools → Performance panel → record page load
   Identify: what IS the LCP element? (image, text, banner?)
   Why is it loading at 3s? (blocked by render-blocking resources? slow server?)

2. IF LCP ELEMENT IS AN IMAGE:
   → Add <link rel="preload"> in <head>
   → Add fetchpriority="high" attribute
   → Ensure the image is on a fast CDN
   → Use next-gen formats (WebP/AVIF)
   → Correct dimensions (no scaling in browser)
   → CDN with global PoPs for geographic proximity

3. IF LCP ELEMENT IS TEXT:
   → Likely blocked by render-blocking CSS or fonts
   → Inline critical CSS (above-the-fold styles)
   → font-display: swap for fonts
   → Preconnect to font CDN origin

4. GLOBAL IMPROVEMENTS:
   → Reduce TTFB (server, CDN, edge functions)
   → Eliminate render-blocking resources
   → Code splitting (defer non-critical JS)
   → Streaming SSR for earlier content delivery
```

---

## 9. Component API Design Questions

### Q: "How do you design a component API? What makes a good API?"

```
PRINCIPLES OF GOOD COMPONENT APIS:

1. PRINCIPLE OF LEAST SURPRISE:
   <Button onClick={handler}>Label</Button>
   Consumers should be able to predict behavior from the API alone.
   Avoid surprising side effects, non-standard behavior.

2. OPEN/CLOSED PRINCIPLE:
   Open for extension, closed for modification.
   Accept className, style, and ...rest for customization
   without requiring the component to change for each use case.

3. CONTROLLED AND UNCONTROLLED MODES:
   Support defaultValue (uncontrolled) and value+onChange (controlled).
   Consumers pick the model that fits their needs.

4. COMPOSITION OVER CONFIGURATION:
   Instead of: <Select renderOption={(opt) => <div>{opt.label}</div>} />
   Prefer:     <Select.Option key={opt.id}>{opt.label}</Select.Option>
   Composition: extensible without adding new props for each use case.

5. PROGRESSIVE DISCLOSURE:
   Simple usage should be simple, complex usage should be possible.
   <Dialog> works with defaults
   <Dialog size="xl" closeOnOverlay={false} onClose={handler}> for customization

6. EXPLICIT OVER IMPLICIT:
   Make required configuration explicit in props.
   Avoid "magic" that works only in specific contexts.

7. FORWARDING REFS:
   Any component that wraps a DOM element should forwardRef.
   Consumers may need focus(), scroll(), measure() access.
```

---

## 10. Architecture Tradeoff Questions

### Q: "How would you choose between client-side rendering (CSR), server-side rendering (SSR), and static generation (SSG)?"

```
DECISION MATRIX:

CSR (React SPA):
  Use when: Highly interactive dashboard, authenticated app, real-time data
  Pros: Fast subsequent navigation, rich interactivity, simpler infrastructure
  Cons: Slow initial load, poor SEO, blank page until JS loads
  Example: Admin panels, internal tools, SaaS dashboards

SSR:
  Use when: Dynamic content that changes per-user or per-request
  Pros: Fast FCP, good SEO, always fresh data
  Cons: Server load per request, more complex infrastructure, TTFB affected by server speed
  Example: E-commerce product pages, personalized feeds

SSG (Static Generation):
  Use when: Content changes infrequently, same for all users
  Pros: Fastest possible load (served from CDN), scales to any traffic, cheapest
  Cons: Requires rebuild for content changes (ISR mitigates this)
  Example: Marketing sites, documentation, blogs

ISR (Incremental Static Regeneration):
  Use when: Content changes periodically but CDN scale is needed
  Pros: CDN-scale with reasonable freshness (regenerate every N minutes)
  Example: News sites, product catalogs

HYBRID (Next.js App Router):
  Page-level and component-level decision
  Static: homepage, about page
  SSR: product pages (SEO + dynamic pricing)
  CSR: interactive widgets, user-specific sections
  RSC: server-side data fetching without client bundle cost
```
