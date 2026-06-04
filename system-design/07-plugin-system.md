# 07 — Plugin Systems

> **"A plugin system is the recognition that you cannot predict all the ways your software will need to extend. Instead of trying, you define the extension points — and let others fill them in."**

Plugin systems allow code to be extended without modifying the core. They're the architecture behind text editors that support thousands of extensions, build tools with hundreds of plugins, and SaaS platforms where customers customize their own experience. Understanding how to design extension points, plugin contracts, lifecycle management, and isolation is the difference between a system that stays flexible and one that accretes ad-hoc conditional logic until nobody dares change it.

---

## 📚 Table of Contents

1. [What a Plugin System Is](#1-what-a-plugin-system-is)
2. [Extension Point Design](#2-extension-point-design)
3. [Plugin Contract (Interface)](#3-plugin-contract-interface)
4. [Plugin Registry and Lifecycle](#4-plugin-registry-and-lifecycle)
5. [Hook-Based Extension Points](#5-hook-based-extension-points)
6. [Slot-Based UI Extension Points](#6-slot-based-ui-extension-points)
7. [Middleware Pattern](#7-middleware-pattern)
8. [Plugin Communication](#8-plugin-communication)
9. [Plugin Isolation and Security](#9-plugin-isolation-and-security)
10. [Plugin Configuration and Settings](#10-plugin-configuration-and-settings)
11. [Versioning and Compatibility](#11-versioning-and-compatibility)
12. [Real-World Examples](#12-real-world-examples)
13. [Good Practices](#13-good-practices)
14. [Bad Practices](#14-bad-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview-Level Explanation](#16-interview-level-explanation)
17. [Exercises](#17-exercises)

---

## 1. What a Plugin System Is

A plugin system consists of three parts:

```
CORE:
  The base application with defined extension points.
  Provides the plugin API — what plugins can do.
  Manages plugin lifecycle (register, init, destroy).
  Doesn't know what plugins will be installed.

EXTENSION POINTS:
  Specific, intentional places where behavior can be customized.
  Named, versioned, and documented.
  Examples: "before request", "render toolbar", "validate form field"

PLUGINS:
  Modules that attach to one or more extension points.
  Self-contained: can be added or removed without touching core.
  Know about the core API but the core doesn't know about them.
```

```
Without plugin system (rigid):
  Core → direct conditional logic
  if (featureFlag.analytics) { trackEvent(event); }
  if (featureFlag.sentry) { captureError(error); }
  if (featureFlag.gtm) { pushToDataLayer(event); }
  → Core grows with every integration. Modifications everywhere.

With plugin system (extensible):
  Core calls: hooks.call('event:track', data)
  Analytics plugin: hooks.on('event:track', sendToAnalytics)
  Sentry plugin:    hooks.on('error:capture', sendToSentry)
  GTM plugin:       hooks.on('event:track', pushToGTM)
  → Core never changes. Integrations are plugins.
```

---

## 2. Extension Point Design

Extension points must be designed deliberately — they are the public API of your plugin system.

### Categories of Extension Points

```
LIFECYCLE HOOKS:
  Fire at specific moments in application lifecycle.
  Plugins observe (side effects only, no return value used).
  Examples: app:init, route:change, user:login, error:catch

DATA TRANSFORMATION HOOKS:
  Allow plugins to modify data flowing through the system.
  Plugins return modified data (or the original).
  Examples: request:before-send, response:transform, form:validate

UI SLOTS:
  Named regions in the UI where plugins can render components.
  Examples: toolbar:right, sidebar:top, card:footer, modal:actions

COMMAND OVERRIDES:
  Plugins can replace or wrap core commands.
  Examples: override the default save behavior, add confirmation dialogs
```

### Extension Point Naming Convention

```typescript
// Convention: 'domain:action' or 'domain:position'
// Lifecycle: 'app:init', 'app:ready', 'app:destroy'
// Route:     'route:before-change', 'route:after-change'
// Request:   'request:before-send', 'request:after-receive', 'request:error'
// UI:        'toolbar:left', 'sidebar:bottom', 'table:row-actions'
// Data:      'user:before-save', 'user:after-load'
// Form:      'form:before-submit', 'form:field-render', 'form:validate'
```

---

## 3. Plugin Contract (Interface)

Every plugin must conform to a typed contract:

```typescript
// The Plugin interface — all plugins implement this
interface Plugin {
  // Required metadata
  id: string; // unique identifier
  name: string; // human-readable name
  version: string; // semver

  // Optional lifecycle
  install?: (api: PluginAPI) => void | Promise<void>;
  uninstall?: () => void | Promise<void>;

  // Optional dependencies
  requires?: string[]; // other plugin IDs this depends on

  // Optional config schema
  configSchema?: unknown; // JSON Schema or Zod schema for settings
}

// The PluginAPI: what the core exposes to plugins
interface PluginAPI {
  // Hook system
  hooks: HookSystem;

  // UI slots
  slots: SlotSystem;

  // App services plugins can use
  services: {
    http: HttpClient;
    router: Router;
    storage: Storage;
    logger: Logger;
  };

  // Settings access
  getConfig: <T>(key: string) => T | undefined;
  setConfig: <T>(key: string, value: T) => void;
}
```

### A Complete Plugin

```typescript
// plugins/analytics/index.ts — a self-contained plugin
import type { Plugin, PluginAPI } from "@/core/plugin";

const analyticsPlugin: Plugin = {
  id: "analytics",
  name: "Analytics",
  version: "1.2.0",

  install(api: PluginAPI) {
    const trackingId = api.getConfig<string>("analytics.trackingId");
    if (!trackingId) {
      api.services.logger.warn("[Analytics] No tracking ID configured.");
      return;
    }

    // Initialize analytics SDK
    initAnalyticsSDK(trackingId);

    // Register hooks
    api.hooks.on("app:ready", () => {
      analytics.pageview(window.location.pathname);
    });

    api.hooks.on("route:after-change", ({ to }) => {
      analytics.pageview(to.path);
    });

    api.hooks.on("user:login-success", ({ user }) => {
      analytics.identify(user.id, {
        plan: user.plan,
        created_at: user.createdAt,
      });
    });

    api.hooks.on("app:error", ({ message, severity }) => {
      if (severity === "high") {
        analytics.trackEvent("error", { message, severity });
      }
    });

    // UI contribution: add analytics status to debug panel
    api.slots.register("debug:panel", AnalyticsDebugPanel);
  },

  uninstall() {
    analytics.reset();
  },
};

export default analyticsPlugin;
```

---

## 4. Plugin Registry and Lifecycle

```typescript
// core/PluginRegistry.ts
class PluginRegistry {
  #plugins = new Map<string, Plugin>();
  #installed = new Map<string, { plugin: Plugin; api: PluginAPI }>();
  #core: CoreAPI;

  constructor(core: CoreAPI) {
    this.#core = core;
  }

  // Register without installing (at app build time)
  register(plugin: Plugin): this {
    if (this.#plugins.has(plugin.id)) {
      console.warn(`[PluginRegistry] Plugin '${plugin.id}' already registered`);
      return this;
    }
    this.#plugins.set(plugin.id, plugin);
    return this; // chainable
  }

  // Install a registered plugin
  async install(pluginId: string): Promise<void> {
    const plugin = this.#plugins.get(pluginId);
    if (!plugin) throw new Error(`Plugin '${pluginId}' not registered`);
    if (this.#installed.has(pluginId)) return; // already installed

    // Check dependencies
    for (const dep of plugin.requires ?? []) {
      if (!this.#installed.has(dep)) {
        throw new Error(
          `Plugin '${pluginId}' requires '${dep}' to be installed first`,
        );
      }
    }

    // Create plugin-scoped API
    const api = this.#createPluginAPI(plugin);

    try {
      await plugin.install?.(api);
      this.#installed.set(pluginId, { plugin, api });
      console.info(`[PluginRegistry] Plugin '${plugin.name}' installed`);
    } catch (err) {
      console.error(
        `[PluginRegistry] Failed to install '${plugin.name}':`,
        err,
      );
      throw err;
    }
  }

  // Install all registered plugins in dependency order
  async installAll(): Promise<void> {
    const sorted = this.#topologicalSort();
    for (const pluginId of sorted) {
      await this.install(pluginId);
    }
  }

  async uninstall(pluginId: string): Promise<void> {
    const entry = this.#installed.get(pluginId);
    if (!entry) return;

    // Check if other installed plugins depend on this one
    for (const [id, { plugin }] of this.#installed) {
      if (id !== pluginId && plugin.requires?.includes(pluginId)) {
        throw new Error(
          `Cannot uninstall '${pluginId}': plugin '${id}' depends on it`,
        );
      }
    }

    await entry.plugin.uninstall?.();
    this.#installed.delete(pluginId);
  }

  // Create a plugin-scoped API (prevents plugins accessing each other's data)
  #createPluginAPI(plugin: Plugin): PluginAPI {
    return {
      hooks: this.#core.hooks.createNamespaced(plugin.id),
      slots: this.#core.slots.createNamespaced(plugin.id),
      services: this.#core.services,
      getConfig: (key) => this.#core.config.get(`${plugin.id}.${key}`),
      setConfig: (key, value) =>
        this.#core.config.set(`${plugin.id}.${key}`, value),
    };
  }

  // Topological sort for dependency-ordered installation
  #topologicalSort(): string[] {
    const visited = new Set<string>();
    const result: string[] = [];

    const visit = (id: string) => {
      if (visited.has(id)) return;
      visited.add(id);
      const plugin = this.#plugins.get(id);
      plugin?.requires?.forEach((dep) => visit(dep));
      result.push(id);
    };

    for (const id of this.#plugins.keys()) visit(id);
    return result;
  }

  get installedPlugins() {
    return [...this.#installed.values()].map((e) => e.plugin);
  }
}
```

---

## 5. Hook-Based Extension Points

Hooks are the most flexible extension mechanism. The core calls hooks at defined points; plugins attach to hooks.

```typescript
// core/HookSystem.ts
type HookHandler<TArgs = void, TReturn = void> = (args: TArgs) => TReturn;
type AsyncHookHandler<TArgs = void, TReturn = void> = (
  args: TArgs,
) => Promise<TReturn> | TReturn;

class HookSystem {
  #hooks = new Map<
    string,
    { handler: Function; priority: number; namespace: string }[]
  >();

  // Register a hook handler
  on<T = void>(
    hookName: string,
    handler: HookHandler<T>,
    priority = 10,
  ): () => void {
    return this.#register(hookName, handler, priority);
  }

  // Register an async hook handler
  onAsync<T = void>(
    hookName: string,
    handler: AsyncHookHandler<T>,
    priority = 10,
  ): () => void {
    return this.#register(hookName, handler, priority);
  }

  #register(
    hookName: string,
    handler: Function,
    priority: number,
    namespace = "",
  ): () => void {
    if (!this.#hooks.has(hookName)) {
      this.#hooks.set(hookName, []);
    }

    const entry = { handler, priority, namespace };
    const handlers = this.#hooks.get(hookName)!;
    handlers.push(entry);
    handlers.sort((a, b) => a.priority - b.priority); // lower number = higher priority

    return () => {
      const i = handlers.indexOf(entry);
      if (i !== -1) handlers.splice(i, 1);
    };
  }

  // FIRE: call all handlers (no return value collection)
  fire<T = void>(hookName: string, args: T): void {
    const handlers = this.#hooks.get(hookName) ?? [];
    for (const { handler } of handlers) {
      try {
        handler(args);
      } catch (err) {
        console.error(`[Hooks] Error in '${hookName}' handler:`, err);
      }
    }
  }

  // FILTER: each handler can modify the value (waterfall)
  filter<T>(hookName: string, value: T): T {
    const handlers = this.#hooks.get(hookName) ?? [];
    return handlers.reduce((current, { handler }) => {
      try {
        const result = handler(current);
        return result !== undefined ? result : current;
      } catch (err) {
        console.error(`[Hooks] Error in '${hookName}' filter:`, err);
        return current;
      }
    }, value);
  }

  // ASYNC FILTER: async waterfall
  async asyncFilter<T>(hookName: string, value: T): Promise<T> {
    const handlers = this.#hooks.get(hookName) ?? [];
    let current = value;
    for (const { handler } of handlers) {
      try {
        const result = await handler(current);
        if (result !== undefined) current = result;
      } catch (err) {
        console.error(`[Hooks] Error in '${hookName}' async filter:`, err);
      }
    }
    return current;
  }

  // Create a namespaced version for a plugin
  createNamespaced(namespace: string): HookSystem {
    const scoped = new HookSystem();
    // Delegate to parent with namespace tracking
    scoped.#hooks = this.#hooks; // share the same hooks map
    return scoped;
  }
}
```

### Using Hooks in the Core

```typescript
// core/RequestManager.ts
class RequestManager {
  #hooks: HookSystem;

  constructor(hooks: HookSystem) {
    this.#hooks = hooks;
  }

  async send<T>(config: RequestConfig): Promise<T> {
    // FILTER hook: plugins can modify the request before sending
    const finalConfig = await this.#hooks.asyncFilter(
      "request:before-send",
      config,
    );

    try {
      const response = await fetch(finalConfig.url, finalConfig);
      const data = await response.json();

      // FILTER hook: plugins can transform the response
      const finalData = await this.#hooks.asyncFilter(
        "request:transform-response",
        {
          data,
          response,
          config: finalConfig,
        },
      );

      // FIRE hook: notify plugins of successful request
      this.#hooks.fire("request:success", {
        data: finalData.data,
        config: finalConfig,
      });

      return finalData.data as T;
    } catch (error) {
      // FILTER hook: plugins can handle/transform errors
      const handled = await this.#hooks.asyncFilter("request:error", {
        error,
        config: finalConfig,
      });
      if (!handled.rethrow) return handled.fallback as T;
      throw error;
    }
  }
}
```

---

## 6. Slot-Based UI Extension Points

Slots allow plugins to inject React components into defined regions of the UI.

```typescript
// core/SlotSystem.ts
import { ComponentType } from 'react';

interface SlotEntry {
  component: ComponentType<any>;
  priority:  number;
  namespace: string;
  props?:    Record<string, unknown>;
}

class SlotSystem {
  #slots      = new Map<string, SlotEntry[]>();
  #listeners  = new Set<() => void>();

  register(
    slotName:  string,
    component: ComponentType<any>,
    options:   { priority?: number; props?: Record<string, unknown> } = {},
    namespace = ''
  ): () => void {
    if (!this.#slots.has(slotName)) this.#slots.set(slotName, []);

    const entry: SlotEntry = {
      component,
      priority: options.priority ?? 10,
      namespace,
      props: options.props,
    };

    const entries = this.#slots.get(slotName)!;
    entries.push(entry);
    entries.sort((a, b) => a.priority - b.priority);

    this.#notify(); // trigger re-render of slot consumers

    return () => {
      const i = entries.indexOf(entry);
      if (i !== -1) entries.splice(i, 1);
      this.#notify();
    };
  }

  getComponents(slotName: string): SlotEntry[] {
    return this.#slots.get(slotName) ?? [];
  }

  #notify() {
    this.#listeners.forEach(fn => fn());
  }

  subscribe(listener: () => void): () => void {
    this.#listeners.add(listener);
    return () => this.#listeners.delete(listener);
  }
}

// The Slot React component: renders all registered components for a slot
function Slot({ name, ...sharedProps }: { name: string; [key: string]: unknown }) {
  const slots    = useSlotSystem(); // access SlotSystem from context
  const [, force] = useReducer(x => x + 1, 0);

  useEffect(() => {
    return slots.subscribe(force); // re-render when plugins register/unregister
  }, [slots, force]);

  const entries = slots.getComponents(name);

  if (entries.length === 0) return null;

  return (
    <>
      {entries.map(({ component: Component, namespace, props }, i) => (
        <ErrorBoundary key={`${namespace}-${i}`} fallback={null}>
          <Component {...sharedProps} {...props} />
        </ErrorBoundary>
      ))}
    </>
  );
}
```

### Using Slots in the Core UI

```tsx
// Core application layout with slot-based extension points
function AppHeader() {
  const { user } = useAuth();
  return (
    <header className="app-header">
      <Logo />

      {/* Core navigation */}
      <nav>{/* ... */}</nav>

      {/* Plugin extension point: left side of toolbar */}
      <Slot name="toolbar:left" user={user} />

      <div className="spacer" />

      {/* Plugin extension point: right side of toolbar */}
      <Slot name="toolbar:right" user={user} />

      {/* Core avatar */}
      <UserAvatar user={user} />
    </header>
  );
}

// Plugin that adds to the toolbar:
// plugins/notifications/index.ts
api.slots.register("toolbar:right", NotificationBell, { priority: 5 });
api.slots.register("toolbar:right", SearchButton, { priority: 10 });

// Result: toolbar shows [NotificationBell] [SearchButton] without core knowing about them
```

---

## 7. Middleware Pattern

Middleware wraps core operations, allowing plugins to intercept and modify behavior in sequence.

```typescript
// Koa-style middleware: each middleware calls next() to continue the chain
type Middleware<T> = (context: T, next: () => Promise<void>) => Promise<void>;

class MiddlewarePipeline<T> {
  #middlewares: Middleware<T>[] = [];

  use(middleware: Middleware<T>): this {
    this.#middlewares.push(middleware);
    return this;
  }

  async execute(
    context: T,
    finalHandler: (ctx: T) => Promise<void>,
  ): Promise<void> {
    const dispatch = (index: number): Promise<void> => {
      if (index === this.#middlewares.length) {
        return finalHandler(context);
      }
      const middleware = this.#middlewares[index];
      return middleware(context, () => dispatch(index + 1));
    };

    return dispatch(0);
  }
}

// Usage: HTTP request pipeline
interface RequestContext {
  method: string;
  url: string;
  headers: Record<string, string>;
  body?: unknown;
  response?: Response;
}

const requestPipeline = new MiddlewarePipeline<RequestContext>();

// Auth middleware (added by auth plugin)
requestPipeline.use(async (ctx, next) => {
  const token = getAuthToken();
  if (token) {
    ctx.headers["Authorization"] = `Bearer ${token}`;
  }
  await next(); // continue to next middleware
});

// Logging middleware (added by logging plugin)
requestPipeline.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  console.log(
    `${ctx.method} ${ctx.url} → ${ctx.response?.status} (${Date.now() - start}ms)`,
  );
});

// Retry middleware (added by resilience plugin)
requestPipeline.use(async (ctx, next) => {
  let attempts = 0;
  while (attempts < 3) {
    try {
      await next();
      break;
    } catch (err) {
      attempts++;
      if (attempts === 3) throw err;
      await new Promise((r) => setTimeout(r, 1000 * attempts));
    }
  }
});

// Final handler: the actual fetch
await requestPipeline.execute(
  { method: "GET", url: "/api/users", headers: {} },
  async (ctx) => {
    ctx.response = await fetch(ctx.url, { headers: ctx.headers });
  },
);
```

---

## 8. Plugin Communication

Plugins should not directly import each other. They communicate through the core.

### Via Events

```typescript
// Plugin A publishes an event
api.hooks.fire("payment:method-selected", { method: "stripe", amount: 99.99 });

// Plugin B subscribes to it
api.hooks.on("payment:method-selected", ({ method, amount }) => {
  if (method === "stripe") initStripeWidget(amount);
});
```

### Via Shared Services

```typescript
// Core exposes a shared data store accessible to all plugins
api.services.store.set("cart:items", items);

// Another plugin reads it
const items = api.services.store.get<CartItem[]>("cart:items");
```

### Via Dependencies (Explicit)

```typescript
// Plugin B explicitly depends on Plugin A
const pluginB: Plugin = {
  id: "stripe-payment",
  requires: ["payment-core"], // must be installed first

  install(api) {
    // Payment-core plugin registered a service
    const paymentService = api.services.get<PaymentService>(
      "payment-core:service",
    );
    paymentService.registerProvider("stripe", StripeProvider);
  },
};
```

---

## 9. Plugin Isolation and Security

Plugins can be malicious or buggy. Design isolation accordingly.

### Error Isolation

```typescript
// Wrap plugin code in try/catch to prevent one bad plugin crashing the app
class SafeHookSystem extends HookSystem {
  override fire<T>(hookName: string, args: T): void {
    const handlers = this.getHandlers(hookName);
    for (const { handler, namespace } of handlers) {
      try {
        handler(args);
      } catch (err) {
        console.error(`[Plugin:${namespace}] Error in '${hookName}':`, err);
        // Report to error monitoring but don't rethrow
        errorMonitor.capture(err, { plugin: namespace, hook: hookName });
      }
    }
  }
}
```

### API Surface Control

```typescript
// Don't expose the full core to plugins — create a minimal API surface
function createPluginAPI(plugin: Plugin, core: Core): PluginAPI {
  return {
    // Only expose what plugins need
    hooks: core.hooks.createNamespaced(plugin.id),
    slots: core.slots.createNamespaced(plugin.id),

    // Expose services, but not the ability to override core services
    services: {
      http: core.http.createInstance(), // isolated instance
      logger: core.logger.withPrefix(`[${plugin.name}]`),
      storage: core.storage.createNamespace(plugin.id), // scoped storage
    },

    getConfig: (key) => core.config.get(`plugins.${plugin.id}.${key}`),

    // NO access to:
    // - Other plugins' APIs
    // - Core internals
    // - Window or document directly (controlled via services)
    // - Other plugins' namespaced storage
  };
}
```

### Sandboxed Plugins (Worker-based)

```typescript
// For untrusted third-party plugins: run in a Web Worker
class SandboxedPlugin implements Plugin {
  #worker: Worker | null = null;

  install(api: PluginAPI) {
    this.#worker = new Worker(this.pluginUrl, { type: "module" });

    // Expose limited API to worker via message passing
    this.#worker.onmessage = ({ data }) => {
      switch (data.type) {
        case "HOOK_ON":
          api.hooks.on(data.hookName, (args) => {
            this.#worker!.postMessage({
              type: "HOOK_FIRED",
              hookName: data.hookName,
              args,
            });
          });
          break;
        case "SLOT_REGISTER":
          // Register a remote component (rendered via portal)
          api.slots.register(
            data.slotName,
            RemoteComponentBridge(this.#worker!),
          );
          break;
      }
    };
  }

  uninstall() {
    this.#worker?.terminate();
  }
}
```

---

## 10. Plugin Configuration and Settings

```typescript
// Define plugin settings schema
const analyticsPlugin: Plugin = {
  id: 'analytics',

  configSchema: z.object({
    trackingId:    z.string().min(1, 'Tracking ID is required'),
    anonymizeIp:   z.boolean().default(true),
    excludePaths:  z.array(z.string()).default(['/admin/**']),
    sampleRate:    z.number().min(0).max(1).default(1.0),
  }),

  install(api) {
    // Config is validated before install is called
    const config = api.getConfig<AnalyticsConfig>('settings');
    initAnalytics({
      id:          config.trackingId,
      anonymizeIp: config.anonymizeIp,
      sampleRate:  config.sampleRate,
    });
  },
};

// Plugin settings UI — generated from schema
function PluginSettingsPanel({ pluginId }: { pluginId: string }) {
  const plugin = usePlugin(pluginId);
  const { data: settings, mutate } = usePluginSettings(pluginId);

  if (!plugin.configSchema) return <p>No settings available.</p>;

  return (
    <FormRenderer
      schema={buildFormSchema(plugin.configSchema)} // config-driven form!
      defaultValues={settings}
      onSubmit={mutate}
    />
  );
}
```

---

## 11. Versioning and Compatibility

```typescript
// Plugin declares its minimum required core version
const myPlugin: Plugin = {
  id: "my-plugin",
  version: "2.1.0",

  // Semver range for required core version
  coreVersion: ">=2.0.0 <3.0.0",

  install(api) {
    // Check API version compatibility at install time
    if (!api.isCompatible(">=2.0.0")) {
      throw new Error("This plugin requires core version 2.0+");
    }
    // ...
  },
};

// Core validates compatibility before installing
function validateCompatibility(plugin: Plugin, coreVersion: string): void {
  if (plugin.coreVersion) {
    const semver = require("semver");
    if (!semver.satisfies(coreVersion, plugin.coreVersion)) {
      throw new Error(
        `Plugin '${plugin.id}' requires core ${plugin.coreVersion}` +
          ` but core is ${coreVersion}`,
      );
    }
  }
}
```

### Deprecation Strategy for Extension Points

```typescript
// When removing an extension point: deprecate first, then remove
hooks.fire("toolbar:action", data); // new
hooks.fire("toolbar:button-click", data); // DEPRECATED: removed in v4

// Or: forward old hook to new hook during transition
hooks.on("toolbar:button-click", (data) => {
  console.warn(
    "[Core] toolbar:button-click is deprecated. Use toolbar:action instead.",
  );
  hooks.fire("toolbar:action", data);
});
```

---

## 12. Real-World Examples

### Text Editor Plugin System (VS Code–like)

```typescript
// Core editor exposes extension points
const editor = new Editor();

// Language plugin: adds syntax highlighting
editor.plugins.register({
  id: "language-javascript",
  install(api) {
    api.hooks.on("editor:parse", (content) => ({
      ...content,
      tokens: jsTokenizer.tokenize(content.text),
    }));
    api.slots.register("gutter:right", JsLintIndicator);
    api.commands.register("format-document", () =>
      jsFormatter.format(editor.content),
    );
  },
});

// Theme plugin: changes appearance
editor.plugins.register({
  id: "theme-dark",
  install(api) {
    api.hooks.on("app:ready", () => document.body.classList.add("theme-dark"));
    api.hooks.on("app:destroy", () =>
      document.body.classList.remove("theme-dark"),
    );
  },
});
```

### Build Tool Plugin System (Vite–like)

```typescript
// Every Vite plugin is an object with hooks
const myVitePlugin = (): VitePlugin => ({
  name: "my-plugin",

  // Transform hook: process files
  transform(code, id) {
    if (!id.endsWith(".myext")) return null;
    return { code: transformCode(code), map: null };
  },

  // Load hook: intercept file loads
  load(id) {
    if (id === "virtual:my-module") {
      return "export const value = 42;";
    }
  },

  // Config hook: modify build config
  config(config) {
    return { ...config, resolve: { ...config.resolve /* additions */ } };
  },
});
```

---

## 13. Good Practices

### ✅ Define extension points based on what you actually need

```typescript
// ✅ Add extension points when you have real uses
// Start with the simplest thing that works:
//   1. Write the code directly
//   2. Extract a hook when the second use case appears
//   3. Define an extension point API when the third case appears

// Don't design a full plugin system speculatively.
// Add extension points as you prove you need them.
```

### ✅ Error boundaries around every plugin contribution

```tsx
// ✅ Plugins are isolated — one broken plugin doesn't break the UI
{
  entries.map((entry, i) => (
    <ErrorBoundary
      key={`${entry.namespace}-${i}`}
      fallback={<PluginErrorFallback plugin={entry.namespace} />}
    >
      <entry.component {...sharedProps} />
    </ErrorBoundary>
  ));
}
```

### ✅ Log plugin lifecycle events for debugging

```typescript
// ✅ Make plugin system observable
registry.on("plugin:installed", ({ id }) =>
  console.info(`✓ Plugin '${id}' installed`),
);
registry.on("plugin:failed", ({ id, error }) =>
  console.error(`✗ Plugin '${id}' failed:`, error),
);
registry.on("plugin:uninstalled", ({ id }) =>
  console.info(`- Plugin '${id}' uninstalled`),
);
```

---

## 14. Bad Practices

### ❌ Plugins that modify core internals

```typescript
// ❌ Plugin monkeypatches core — fragile and untestable
install(api) {
  const originalSend = api.services.http.send;
  api.services.http.send = async (config) => {
    config.headers['X-Plugin'] = 'true';
    return originalSend(config); // mutates the shared HTTP client
  };
}

// ✅ Use the provided hook
api.hooks.on('request:before-send', (config) => ({
  ...config,
  headers: { ...config.headers, 'X-Plugin': 'true' },
}));
```

### ❌ Circular plugin dependencies

```typescript
// ❌ A requires B, B requires A — deadlock
const pluginA = { id: "a", requires: ["b"] };
const pluginB = { id: "b", requires: ["a"] };
// Registry will detect this during topological sort and throw
```

### ❌ Side effects in plugin register() (before install)

```typescript
// ❌ Network calls during registration — too early
const myPlugin = {
  id: 'my-plugin',
  version: '1.0.0',
  config: fetch('/api/plugin-config').then(r => r.json()), // ❌ side effect at module level
  install(api) { ... },
};

// ✅ All side effects in install()
const myPlugin = {
  id: 'my-plugin',
  version: '1.0.0',
  async install(api) {
    const config = await fetch('/api/plugin-config').then(r => r.json()); // ✅
    // ...
  },
};
```

---

## 15. Common Mistakes

### Mistake 1 — Not providing a way to uninstall/clean up

```typescript
// ❌ Install adds event listeners with no way to remove them
install(api) {
  document.addEventListener('keydown', handleKeydown);
  // Never removed when plugin is uninstalled
}

// ✅ Collect cleanup functions
install(api) {
  const cleanups = [
    api.hooks.on('app:ready', initPlugin),
    api.slots.register('toolbar:left', PluginButton),
  ];
  // Return cleanup function
  return () => cleanups.forEach(fn => fn());
}
```

### Mistake 2 — Plugin that assumes it's the only one

```typescript
// ❌ Plugin sets a value, assuming no other plugin will touch it
api.hooks.filter("user:display-name", () => "Custom Name"); // replaces, doesn't compose

// ✅ Plugin transforms the existing value
api.hooks.filter("user:display-name", (name) => `${name} (verified)`); // composes
```

### Mistake 3 — Exposing too much in the plugin API

```typescript
// ❌ Full core access — plugins can do anything
api.core.router.navigate("/hack");
api.core.store.dispatch({ type: "RESET_EVERYTHING" });
api.core.eventBus.unsubscribeAll(); // would break other plugins

// ✅ Minimal capability principle: expose only what plugins need
// Plugins get a scoped API that limits potential damage
```

---

## 16. Interview-Level Explanation

> **"How would you design a plugin system for a frontend application?"**

**Strong answer:**

> "A plugin system has three parts: the core, the extension points, and the plugins themselves. The core defines named extension points — specific places where behavior can be customized. Plugins install by attaching handlers to these extension points. The core doesn't know what plugins will exist; plugins don't need to modify core code.
>
> The two main types of extension points are hooks and slots. Hooks are lifecycle events or data transformations — 'before request is sent,' 'after user logs in,' 'validate form field.' Hooks can be fire-and-forget (the core calls all handlers, ignores return values) or filter-style (each handler can modify data that flows to the next). Slots are UI extension points — named regions where plugins can inject React components. A toolbar might have a 'toolbar:right' slot; notification and search plugins each register a component into that slot without the toolbar knowing about either.
>
> The plugin contract defines what every plugin must implement: an ID, a name, a version, an `install` function that receives the plugin API, and an `uninstall` cleanup function. The install function receives a scoped API — not full access to the core, but enough: the hook system, the slot system, scoped services, and scoped config storage. This limits what a buggy or malicious plugin can do.
>
> Dependency management comes from a registry that handles topological sorting. If Plugin B requires Plugin A, the registry installs A first. Circular dependencies are detected and rejected.
>
> The critical design discipline is isolation. Every plugin contribution — hook handlers, slot components — runs in an error boundary or try/catch. One failing plugin cannot break the application. Plugin components render in React ErrorBoundaries. Hook handlers are wrapped to catch errors and log them without rethrowing.
>
> Extension points should be designed conservatively — add them when you have a real use case, not speculatively. The moment you define an extension point, you're committing to supporting it. Removing it is a breaking change that affects all plugins that depend on it."

---

## 17. Exercises

### Exercise 1 — Design an admin panel plugin system

Design a plugin system for an admin panel where:

- Teams can add new sections to the admin sidebar
- Teams can add widgets to the dashboard
- Teams can extend user table with custom columns
- The core provides: auth state, HTTP client, notification service

Define: the Plugin interface, 3 extension points, and what the plugin API exposes.

<details>
<summary>Solution</summary>

```typescript
// Plugin interface
interface AdminPlugin {
  id: string;
  name: string;
  version: string;
  requires?: string[];
  install: (api: AdminPluginAPI) => void | (() => void); // returns cleanup fn
}

// Plugin API
interface AdminPluginAPI {
  // Navigation: add sidebar sections
  nav: {
    register(item: {
      id: string;
      label: string;
      icon?: string;
      href: string;
      section?: string; // group in sidebar
      permissions?: string[];
    }): () => void;
  };

  // Dashboard: add widgets
  dashboard: {
    register(widget: {
      id: string;
      component: ComponentType<{ data?: unknown }>;
      title: string;
      defaultSize?: "small" | "medium" | "large";
    }): () => void;
  };

  // User table: add columns
  userTable: {
    addColumn(column: {
      id: string;
      header: string;
      field?: string;
      component?: ComponentType<{ user: User }>;
      width?: number;
      sortable?: boolean;
    }): () => void;
  };

  // Services available to plugins
  services: {
    http: HttpClient; // for API calls
    auth: { getUser: () => User }; // read-only auth access
    notify: (msg: Notification) => void; // show notifications
    logger: Logger;
  };

  // Scoped config
  getConfig: <T>(key: string) => T | undefined;
}

// Extension points:
// 1. NAV: sidebar navigation items
// 2. DASHBOARD: configurable dashboard widgets
// 3. USER_TABLE: extensible user table columns

// Example plugin that adds an "Inventory" section
const inventoryPlugin: AdminPlugin = {
  id: "inventory",
  name: "Inventory Management",
  version: "1.0.0",

  install(api) {
    const cleanups = [
      api.nav.register({
        id: "inventory",
        label: "Inventory",
        icon: "box",
        href: "/admin/inventory",
      }),
      api.dashboard.register({
        id: "low-stock-widget",
        title: "Low Stock Alert",
        component: LowStockWidget,
        defaultSize: "medium",
      }),
      api.userTable.addColumn({
        id: "managed-products",
        header: "Products Managed",
        component: UserProductCount,
        width: 120,
        sortable: true,
      }),
    ];

    return () => cleanups.forEach((fn) => fn());
  },
};
```

</details>

---

## 🔗 Related Topics

- [`system-design/06-event-driven-frontend.md`](./06-event-driven-frontend.md) — Events as plugin communication
- [`system-design/05-config-driven-ui.md`](./05-config-driven-ui.md) — Config-driven UI for plugin settings
- [`patterns/03-command.md`](../patterns/03-command.md) — Command pattern for plugin actions
- [`javascript-core/14-observer-patterns.md`](../javascript-core/14-observer-patterns.md) — Observer pattern behind hooks

---

<div align="center">

**Next:** [`system-design/08-design-tradeoffs.md`](./08-design-tradeoffs.md) →

</div>
