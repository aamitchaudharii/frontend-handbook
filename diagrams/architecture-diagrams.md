# Architecture Diagrams

> **"A diagram earns its place when it communicates in two seconds what would take two paragraphs to write. These are the diagrams worth having on a reference card: the browser rendering pipeline, React's lifecycle, the event loop, request flows, and architectural patterns."**

All diagrams are written in Mermaid and render natively in GitHub, GitLab, Notion, Obsidian, and most modern markdown viewers.

---

## Browser Rendering Pipeline

```mermaid
flowchart LR
  HTML["HTML Parsing\n(DOM construction)"] --> Style["Style Calculation\n(CSSOM)"]
  Style --> Layout["Layout\n(box positions & sizes)"]
  Layout --> Paint["Paint\n(pixel instructions)"]
  Paint --> Composite["Composite\n(layer assembly)"]
  Composite --> Screen["Screen"]

  HTML --> DOM["DOM Tree"]
  Style --> CSSOM["CSSOM Tree"]
  DOM --> RenderTree["Render Tree"]
  CSSOM --> RenderTree
  RenderTree --> Layout

  classDef expensive fill:#fef3c7,stroke:#d97706
  classDef compositor fill:#dcfce7,stroke:#16a34a
  class Layout,Paint expensive
  class Composite compositor
```

---

## CSS Property Animation Cost

```mermaid
flowchart TD
  Prop["CSS Property Changed"] --> Q1{Affects\nlayout?}
  Q1 -->|Yes| Layout["Layout → Paint → Composite\n(most expensive)"]
  Q1 -->|No| Q2{Affects\npaint?}
  Q2 -->|Yes| Paint["Paint → Composite\n(expensive)"]
  Q2 -->|No| Composite["Composite only\n✅ FAST — transform, opacity"]

  Layout --> Examples1["width, height, margin,\npadding, top, left"]
  Paint  --> Examples2["background-color,\nborder, box-shadow"]
  Composite --> Examples3["transform: translate/scale/rotate\nopacity\nfilter (GPU-accelerated)"]

  classDef bad fill:#fef2f2,stroke:#dc2626
  classDef ok fill:#fef3c7,stroke:#d97706
  classDef good fill:#dcfce7,stroke:#16a34a
  class Layout bad
  class Paint ok
  class Composite good
```

---

## The JavaScript Event Loop

```mermaid
sequenceDiagram
  participant CallStack as Call Stack
  participant MicroQueue as Microtask Queue<br/>(Promises, queueMicrotask)
  participant MacroQueue as Macrotask Queue<br/>(setTimeout, setInterval, I/O)
  participant Render as Browser Render

  loop Event Loop Tick
    CallStack->>CallStack: Execute synchronous code
    Note over CallStack: Stack is now empty
    CallStack->>MicroQueue: Drain ALL microtasks<br/>(can queue more microtasks)
    MicroQueue->>CallStack: Run microtask callbacks
    CallStack->>Render: (if frame budget allows)
    Render->>Render: requestAnimationFrame callbacks
    Render->>Render: Paint / Composite
    CallStack->>MacroQueue: Take ONE macrotask
    MacroQueue->>CallStack: Run macrotask callback
  end
```

---

## React Component Lifecycle (Function Component)

```mermaid
flowchart TD
  Mount["MOUNT"] --> Render1["Render\n(call component fn)"]
  Render1 --> Commit1["Commit\n(DOM mutations)"]
  Commit1 --> LayoutEffect1["useLayoutEffect\n(synchronous, before paint)"]
  LayoutEffect1 --> Paint1["Browser Paints"]
  Paint1 --> Effect1["useEffect\n(async, after paint)"]

  Update["UPDATE\n(setState / prop change)"] --> Render2["Render\n(call component fn)"]
  Render2 --> Reconcile["Reconcile\n(diff vs previous)"]
  Reconcile --> Commit2["Commit\n(minimal DOM changes)"]
  Commit2 --> Cleanup2["Cleanup previous\nuseLayoutEffect"]
  Cleanup2 --> LayoutEffect2["useLayoutEffect\n(synchronous)"]
  LayoutEffect2 --> Paint2["Browser Paints"]
  Paint2 --> Cleanup2b["Cleanup previous\nuseEffect"]
  Cleanup2b --> Effect2["useEffect\n(async)"]

  Unmount["UNMOUNT"] --> Cleanup3["Cleanup useLayoutEffect"]
  Cleanup3 --> Cleanup4["Cleanup useEffect"]

  classDef sync fill:#fef3c7,stroke:#d97706
  classDef async fill:#dbeafe,stroke:#2563eb
  class LayoutEffect1,LayoutEffect2 sync
  class Effect1,Effect2 async
```

---

## React Rendering Decision Tree

```mermaid
flowchart TD
  Trigger["setState called\nor prop changed"] --> Fiber["React schedules\na re-render"]
  Fiber --> Bailout{"Is this component\nwrapped in React.memo?"}
  Bailout -->|Yes| PropsCheck{"Did any prop\nchange? (shallow =)"}
  Bailout -->|No| Render
  PropsCheck -->|Yes| Render["Re-render\n(call component fn)"]
  PropsCheck -->|No| Skip["Skip render\n✅ Bailed out"]
  Render --> Reconcile["Reconcile output\nvs. previous tree"]
  Reconcile --> DOMDiff{"Did output\ndiffer from last?"}
  DOMDiff -->|Yes| UpdateDOM["Update real DOM\n(minimal mutations)"]
  DOMDiff -->|No| NoDOMUpdate["No DOM update\n✅ Cheap render"]
```

---

## Token Storage Security Model

```mermaid
flowchart TD
  subgraph Storage["Client-Side Storage Options"]
    LS["localStorage /\nsessionStorage"]
    MEM["In-Memory\n(JS variable)"]
    COOKIE["HttpOnly Cookie"]
  end

  subgraph Threats["Threat Vectors"]
    XSS["XSS Attack\n(injected script)"]
    CSRF["CSRF Attack\n(forged request)"]
  end

  XSS -->|"✅ CAN steal"| LS
  XSS -->|"⚠️ Harder to steal\n(not in global scope)"| MEM
  XSS -->|"❌ CANNOT steal\n(JS can't read)"| COOKIE

  CSRF -->|"❌ NOT auto-sent\n(manual header required)"| LS
  CSRF -->|"❌ NOT auto-sent"| MEM
  CSRF -->|"⚠️ Auto-sent by browser\n(mitigated by SameSite=Strict)"| COOKIE

  subgraph Recommendation["Recommended Architecture"]
    AT["Access Token → Memory\n(short-lived, 15min)"]
    RT["Refresh Token → HttpOnly Cookie\n(SameSite=Strict, long-lived)"]
  end

  classDef bad fill:#fef2f2,stroke:#dc2626
  classDef good fill:#dcfce7,stroke:#16a34a
  classDef warn fill:#fef3c7,stroke:#d97706
  class LS bad
  class MEM warn
  class COOKIE good
```

---

## OAuth 2.0 PKCE Flow (SPA)

```mermaid
sequenceDiagram
  participant User
  participant App as SPA (Browser)
  participant AuthServer as Auth Server
  participant API as Your Backend

  User->>App: Click "Login with Google"
  App->>App: Generate code_verifier + code_challenge (PKCE)
  App->>App: Generate random state (CSRF protection)
  App->>AuthServer: Redirect with code_challenge + state
  AuthServer->>User: Show login / consent screen
  User->>AuthServer: Authenticate & approve
  AuthServer->>App: Redirect back with code + state
  App->>App: Verify state matches stored state
  App->>API: POST /auth/callback { code, code_verifier }
  API->>AuthServer: Exchange code + code_verifier for tokens
  AuthServer->>API: { access_token, refresh_token, id_token }
  API->>App: { accessToken } + Set-Cookie: refresh_token (HttpOnly)
  App->>App: Store access_token in memory
```

---

## HTTP Request Waterfall vs Parallel

```mermaid
gantt
  title Sequential vs Parallel Requests
  dateFormat X
  axisFormat %Lms

  section Sequential (Bad)
  Request 1 : 0, 200
  Request 2 : 200, 400
  Request 3 : 400, 600
  Total (600ms) : milestone, 600, 600

  section Parallel (Good)
  Request 1 : 0, 200
  Request 2 : 0, 300
  Request 3 : 0, 250
  Total (300ms) : milestone, 300, 300
```

---

## State Management Decision Tree

```mermaid
flowchart TD
  Q1["Where does this state\nneed to be accessible?"] --> A1["One component only"]
  Q1 --> A2["A few related components\n(parent + children)"]
  Q1 --> A3["Many unrelated components\nacross the tree"]
  Q1 --> A4["Entire application +\ncomplex interactions"]

  A1 --> S1["useState / useReducer\nin that component"]
  A2 --> S2["useState lifted to\ncommon parent + props"]
  A2 --> S2b["OR: Component composition\n(pass rendered elements)"]
  A3 --> S3["React Context\n(theme, auth, locale)"]
  A4 --> S4["External store\n(Zustand, Redux, Jotai)"]

  Q2["Is this server data?"] --> S5["TanStack Query\n(caching, sync, background refresh)"]

  classDef local fill:#dbeafe,stroke:#2563eb
  classDef lifted fill:#dcfce7,stroke:#16a34a
  classDef context fill:#fef3c7,stroke:#d97706
  classDef external fill:#fce7f3,stroke:#9333ea
  class S1 local
  class S2,S2b lifted
  class S3 context
  class S4,S5 external
```

---

## Micro-Frontend Architecture

```mermaid
flowchart TD
  Shell["Shell Application\n(routing, shared layout, auth)"]

  subgraph MFEs["Micro-Frontends (independently deployed)"]
    MFE1["MFE: Checkout\n(React 18, Team A)"]
    MFE2["MFE: Product Catalog\n(Vue 3, Team B)"]
    MFE3["MFE: User Profile\n(React 17, Team C)"]
  end

  subgraph Shared["Shared Infrastructure"]
    DS["Design System\n(@company/ui-components)"]
    Auth["Auth SDK\n(token management)"]
    Analytics["Analytics\n(event tracking)"]
  end

  Shell --> MFE1
  Shell --> MFE2
  Shell --> MFE3
  MFE1 --> DS
  MFE2 --> DS
  MFE3 --> DS
  MFE1 --> Auth
  MFE2 --> Auth
  MFE3 --> Auth
```

---

## Service Worker Cache Strategies

```mermaid
flowchart TD
  Request["Browser Request"]

  Request --> CF["Cache First\n(static assets: JS, CSS, fonts)"]
  Request --> NF["Network First\n(API calls: fresh data preferred)"]
  Request --> SO["Stale While Revalidate\n(semi-dynamic content)"]
  Request --> NO["Network Only\n(auth, payment, real-time)"]

  CF --> Cache["Cache"]
  Cache -->|Hit| CacheHit["Return cached response\n→ fetch & update cache in background"]
  Cache -->|Miss| Network1["Fetch from network\n→ cache the response"]

  NF --> Network2["Fetch from network"]
  Network2 -->|Success| NetworkSuccess["Return + update cache"]
  Network2 -->|Offline| FallbackCache["Return stale cache\nif available"]

  SO --> CacheParallel["Return stale cache\n(instant)"]
  SO --> NetworkParallel["Fetch from network\n→ update cache\n(for next request)"]
```

---

## React Concurrent Rendering

```mermaid
sequenceDiagram
  participant User
  participant React as React Scheduler
  participant Main as Main Thread

  User->>React: Type in search box (high priority)
  React->>Main: Start rendering search results (low priority)
  Main->>Main: Render in progress...

  User->>React: Type another character (high priority interrupt!)
  React->>Main: INTERRUPT: pause low-priority render
  React->>Main: Handle input update (high priority, urgent)
  Main->>Main: Update input value instantly
  React->>Main: Resume search results render (low priority)
  Main->>Main: Complete render with latest query value
```

---

## WebSocket Connection Lifecycle

```mermaid
stateDiagram-v2
  [*] --> Connecting: connect() called
  Connecting --> Open: WebSocket handshake successful
  Connecting --> Closed: Connection refused / network error
  Open --> Closed: ws.close() or network drop
  Closed --> Reconnecting: Auto-reconnect scheduled\n(exponential backoff)
  Reconnecting --> Connecting: Reconnect attempt
  Open --> Open: Messages flowing\n(onmessage / send)

  note right of Reconnecting: Attempt 1: 1s delay\nAttempt 2: 2s delay\nAttempt 3: 4s delay\nMax: 30s delay
```

---

## Virtualized List — What the Browser Actually Sees

```mermaid
flowchart LR
  subgraph DOM["DOM (only ~15 nodes rendered)"]
    Spacer["Spacer div\nheight: totalHeight\n(gives scrollbar correct size)"]
    Items["Items 47–62\n(absolutely positioned\nwithin spacer)"]
  end

  subgraph Logical["Logical list (10,000 items)"]
    I1["Item 0"]
    I2["Item 1"]
    Dots1["..."]
    I47["Item 47"]
    I62["Item 62 ←viewport end"]
    Dots2["..."]
    I9999["Item 9,999"]
  end

  Viewport["Visible Viewport\n(scrollTop = 2350px)\n(height = 800px)"]

  Viewport --> Items
  I47 -.->|"rendered in"| Items
  I62 -.->|"rendered in"| Items
  Spacer -.->|"height = 9999 × itemHeight"| Logical
```

---

## Error Boundary Placement Strategy

```mermaid
flowchart TD
  Root["ErrorBoundary (root)\nFallback: AppCrashPage\nCatches catastrophic failures"]

  Root --> Layout["AppLayout"]
  Layout --> RouteEB["ErrorBoundary (route)\nFallback: PageError"]
  Layout --> Sidebar["ErrorBoundary (widget)\nFallback: SidebarError"]

  RouteEB --> Page["Page Content"]
  Page --> Widget1["ErrorBoundary (widget)\nFallback: WidgetError"]
  Page --> Widget2["ErrorBoundary (widget)\nFallback: WidgetError"]
  Page --> Widget3["ErrorBoundary (widget)\nFallback: WidgetError"]

  Widget1 --> Chart["RevenueChart"]
  Widget2 --> Table["DataTable"]
  Widget3 --> Feed["ActivityFeed"]

  Sidebar --> Nav["Navigation"]
  Sidebar --> Notif["ErrorBoundary\nFallback: null"]
  Notif --> NotifList["NotificationList"]

  classDef boundary fill:#fef3c7,stroke:#d97706,stroke-width:2px
  class Root,RouteEB,Sidebar,Widget1,Widget2,Widget3,Notif boundary
```

---

## Core Web Vitals — What They Measure

```mermaid
timeline
  title Page Load Timeline → Core Web Vitals
  section Navigation start
    TTFB : Time to First Byte
         : Server response latency
  section Content arrives
    FCP : First Contentful Paint
        : First text or image appears
  section Main content loads
    LCP : Largest Contentful Paint ✅
        : Main content visible
        : Target under 2.5s
  section Interactivity
    INP : Interaction to Next Paint ✅
        : Response to user input
        : Target under 200ms
  section Visual stability
    CLS : Cumulative Layout Shift ✅
        : Visual stability score
        : Target under 0.1
```
