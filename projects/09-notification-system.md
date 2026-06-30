# 09 — Project: Notification System

> **"A notification system touches almost every part of the application but is owned by none of it — any component, anywhere, needs to be able to say 'tell the user X' without knowing or caring how that message gets displayed, queued, prioritized against other messages, or eventually dismissed. This is a deceptively good test of whether your architecture has a clean separation between 'what happened' and 'how the user finds out.'"**

This project guide builds a complete notification system: ephemeral toast notifications, a persistent notification center (bell icon + dropdown), real-time push via WebSocket, priority-based queueing, and the cross-cutting architecture that lets any part of the app trigger a notification without prop drilling or tight coupling.

---

## 📚 What You'll Build

A notification system with: toast notifications (auto-dismissing, stackable, swipeable), a persistent notification center showing notification history with read/unread state, real-time delivery via WebSocket, priority-based display rules (a critical error shouldn't be silently buried under five low-priority toasts), and a clean API any component can call without prop drilling.

---

## Requirements

```
FUNCTIONAL:
  - Toast notifications: appear, stack, auto-dismiss after a duration,
    can be manually dismissed, support action buttons ("Undo")
  - Notification center: bell icon with unread badge, dropdown showing
    notification history, mark as read individually or all at once
  - Real-time delivery: new notifications arrive via WebSocket without
    requiring a page refresh
  - Priority handling: critical notifications (e.g., "payment failed")
    should be visually distinct and not get lost among routine ones

NON-FUNCTIONAL:
  - Any component anywhere in the tree can trigger a toast without prop
    drilling through intermediate components
  - Toast stack must not grow unbounded if many notifications arrive rapidly
  - Notification state (read/unread, history) must persist across page
    refreshes and be synced with the server
```

---

## Architecture Overview

```
COMPONENT TREE:
  <NotificationProvider>          (wraps the app, owns toast + center state)
    <App>
      ... (any component can call useToast() or useNotifications())
    <ToastContainer />            (renders the toast stack, portal to body)
    <NotificationCenter />        (bell icon + dropdown, lives in the header)

ARCHITECTURE PRINCIPLE: notifications are triggered via a Context-provided
function (useToast().show(...)), not by passing callback props down
through the tree. This is the textbook case where Context's "broad,
infrequent... well actually frequent but lightweight" access pattern is
the right tool — MANY unrelated components need to trigger toasts, and
prop drilling a `showToast` function through every intermediate layer
would be exactly the anti-pattern covered in anti-patterns/01-prop-drilling.md.
```

---

## Step 1 — The Toast System (Imperative API, Declarative Rendering)

```typescript
interface Toast {
  id:        string;
  message:   string;
  type:      'info' | 'success' | 'warning' | 'error';
  duration?: number;       // ms, undefined = stays until manually dismissed
  action?:   { label: string; onClick: () => void };
}

const ToastContext = createContext<{
  toasts:  Toast[];
  show:    (toast: Omit<Toast, 'id'>) => string; // returns the toast id
  dismiss: (id: string) => void;
} | null>(null);

function ToastProvider({ children, maxToasts = 5 }: { children: ReactNode; maxToasts?: number }) {
  const [toasts, setToasts] = useState<Toast[]>([]);

  const dismiss = useCallback((id: string) => {
    setToasts(prev => prev.filter(t => t.id !== id));
  }, []);

  const show = useCallback((toast: Omit<Toast, 'id'>) => {
    const id = crypto.randomUUID();
    const newToast: Toast = { id, duration: 4000, ...toast };

    setToasts(prev => {
      // Cap the stack: if at max, drop the OLDEST toast to make room
      // (prevents unbounded growth if notifications arrive faster than
      // they auto-dismiss)
      const next = prev.length >= maxToasts ? prev.slice(1) : prev;
      return [...next, newToast];
    });

    if (newToast.duration) {
      setTimeout(() => dismiss(id), newToast.duration);
    }

    return id;
  }, [dismiss, maxToasts]);

  return (
    <ToastContext.Provider value={{ toasts, show, dismiss }}>
      {children}
    </ToastContext.Provider>
  );
}

function useToast() {
  const ctx = useContext(ToastContext);
  if (!ctx) throw new Error('useToast must be used within ToastProvider');
  return ctx;
}

// Usage from ANY component, anywhere in the tree, no prop drilling:
function SaveButton() {
  const { show } = useToast();

  async function handleSave() {
    try {
      await saveDocument();
      show({ message: 'Document saved', type: 'success' });
    } catch {
      show({
        message: 'Failed to save document',
        type: 'error',
        duration: undefined, // errors stay until manually dismissed
        action: { label: 'Retry', onClick: handleSave },
      });
    }
  }

  return <button onClick={handleSave}>Save</button>;
}
```

**Key decision:** the toast stack has a `maxToasts` cap, and exceeding it drops the OLDEST toast (FIFO), not the newest — if ten things go wrong in rapid succession (e.g., a batch operation with multiple failures), the user sees the five MOST RECENT issues rather than getting stuck looking at five increasingly stale ones while new, possibly more urgent, notifications are silently discarded.

---

## Step 2 — Toast Rendering with Exit Animations

```jsx
function ToastContainer() {
  const { toasts, dismiss } = useToast();

  return createPortal(
    <div className="toast-container" aria-live="polite">
      <AnimatePresence>
        {toasts.map((toast) => (
          <ToastItem
            key={toast.id}
            toast={toast}
            onDismiss={() => dismiss(toast.id)}
          />
        ))}
      </AnimatePresence>
    </div>,
    document.body,
  );
}

function ToastItem({ toast, onDismiss }) {
  return (
    <motion.div
      layout // smoothly animates position when toasts above are dismissed
      initial={{ opacity: 0, y: -20, scale: 0.95 }}
      animate={{ opacity: 1, y: 0, scale: 1 }}
      exit={{ opacity: 0, x: 100 }}
      className={`toast toast--${toast.type}`}
      role={toast.type === "error" ? "alert" : "status"}
    >
      <ToastIcon type={toast.type} />
      <span className="toast-message">{toast.message}</span>
      {toast.action && (
        <button className="toast-action" onClick={toast.action.onClick}>
          {toast.action.label}
        </button>
      )}
      <button className="toast-close" onClick={onDismiss} aria-label="Dismiss">
        ×
      </button>
    </motion.div>
  );
}
```

**Key decision:** the `layout` prop on each toast (Framer Motion) means that when a toast in the middle of the stack is dismissed, the toasts below it smoothly animate UP to fill the gap rather than abruptly jumping — a small detail that prevents the toast stack from feeling jarring during rapid dismissals. The `role` attribute switches between `alert` (interrupts screen readers immediately) for errors and `status` (announced politely, doesn't interrupt) for routine notifications — matching the urgency to the appropriate ARIA live region behavior.

---

## Step 3 — Persistent Notification Center

```typescript
interface Notification {
  id:        string;
  title:     string;
  body:      string;
  priority:  'low' | 'normal' | 'high' | 'critical';
  isRead:    boolean;
  createdAt: string;
  link?:     string; // optional deep link when clicked
}

function useNotificationCenter() {
  const queryClient = useQueryClient();

  const { data: notifications = [] } = useQuery({
    queryKey: ['notifications'],
    queryFn:  () => notificationsApi.list(),
  });

  const unreadCount = notifications.filter(n => !n.isRead).length;

  const markAsReadMutation = useMutation({
    mutationFn: (id: string) => notificationsApi.markRead(id),
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: ['notifications'] });
      const previous = queryClient.getQueryData<Notification[]>(['notifications']);
      queryClient.setQueryData<Notification[]>(['notifications'], (prev = []) =>
        prev.map(n => n.id === id ? { ...n, isRead: true } : n)
      );
      return { previous };
    },
    onError: (err, id, context) => {
      if (context?.previous) queryClient.setQueryData(['notifications'], context.previous);
    },
  });

  const markAllAsReadMutation = useMutation({
    mutationFn: () => notificationsApi.markAllRead(),
    onMutate: async () => {
      await queryClient.cancelQueries({ queryKey: ['notifications'] });
      const previous = queryClient.getQueryData<Notification[]>(['notifications']);
      queryClient.setQueryData<Notification[]>(['notifications'], (prev = []) =>
        prev.map(n => ({ ...n, isRead: true }))
      );
      return { previous };
    },
    onError: (err, vars, context) => {
      if (context?.previous) queryClient.setQueryData(['notifications'], context.previous);
    },
  });

  return {
    notifications,
    unreadCount,
    markAsRead: markAsReadMutation.mutate,
    markAllAsRead: markAllAsReadMutation.mutate,
  };
}

function NotificationCenter() {
  const { notifications, unreadCount, markAsRead, markAllAsRead } = useNotificationCenter();
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className="notification-center">
      <button onClick={() => setIsOpen(o => !o)} aria-label={`Notifications (${unreadCount} unread)`}>
        <BellIcon />
        {unreadCount > 0 && <span className="unread-badge">{unreadCount > 99 ? '99+' : unreadCount}</span>}
      </button>

      {isOpen && (
        <div className="notification-dropdown">
          <div className="notification-header">
            <h3>Notifications</h3>
            {unreadCount > 0 && <button onClick={() => markAllAsRead()}>Mark all as read</button>}
          </div>
          <ul>
            {notifications.map(n => (
              <NotificationItem key={n.id} notification={n} onRead={() => markAsRead(n.id)} />
            ))}
          </ul>
        </div>
      )}
    </div>
  );
}
```

**Key decision:** both `markAsRead` and `markAllAsRead` use the same optimistic-update + rollback pattern (snapshot before, restore on error) established in [`projects/03-kanban-board.md`](./03-kanban-board.md) — marking a notification as read should feel instant, and the rare failure case (network drop) should cleanly revert rather than leave the UI showing a state the server never actually confirmed.

---

## Step 4 — Real-Time Delivery via WebSocket

```typescript
function useRealtimeNotifications() {
  const queryClient = useQueryClient();
  const { show } = useToast();

  useEffect(() => {
    return chatConnection.onMessage((message) => {
      // reusing the connection
      if (message.type !== "notification") return; // manager pattern from
      // projects/01-realtime-chat-application.md
      const notification: Notification = message.payload;

      // 1. Add to the notification center's cached list
      queryClient.setQueryData<Notification[]>(
        ["notifications"],
        (prev = []) => [notification, ...prev],
      );

      // 2. ALSO show as a toast for IMMEDIATE visibility, but only for
      //    high-priority notifications — routine ones just accumulate
      //    quietly in the notification center without interrupting the user
      if (
        notification.priority === "high" ||
        notification.priority === "critical"
      ) {
        show({
          message: notification.title,
          type: notification.priority === "critical" ? "error" : "warning",
          duration: notification.priority === "critical" ? undefined : 6000,
        });
      }
    });
  }, [queryClient, show]);
}
```

**Key decision:** NOT every incoming notification becomes a toast — only high-priority and critical ones interrupt the user's current screen with a toast; routine notifications (e.g., "someone liked your post") are added to the notification center silently, where the unread badge communicates "something happened" without demanding immediate attention. This separation — center for history/low-urgency, toast for immediate/high-urgency — is the core design decision that keeps the toast stack from becoming noise.

---

## Step 5 — Priority-Aware Toast Styling and Stacking Order

```jsx
function ToastContainer() {
  const { toasts } = useToast();

  // Sort critical/error toasts to the TOP of the stack, regardless of
  // arrival order — a critical error shouldn't be buried under three
  // "saved successfully" toasts that arrived moments later
  const sortedToasts = useMemo(() => {
    const priorityOrder = { error: 0, warning: 1, success: 2, info: 3 };
    return [...toasts].sort(
      (a, b) => priorityOrder[a.type] - priorityOrder[b.type],
    );
  }, [toasts]);

  return createPortal(
    <div className="toast-container">
      <AnimatePresence>
        {sortedToasts.map((toast) => (
          <ToastItem key={toast.id} toast={toast} />
        ))}
      </AnimatePresence>
    </div>,
    document.body,
  );
}
```

```css
.toast--error {
  background: #fef2f2;
  border-left: 4px solid #dc2626;
}
.toast--warning {
  background: #fffbeb;
  border-left: 4px solid #d97706;
}
.toast--success {
  background: #f0fdf4;
  border-left: 4px solid #16a34a;
}
.toast--info {
  background: #eff6ff;
  border-left: 4px solid #2563eb;
}
```

---

## Step 6 — Notification Preferences (User Control)

```typescript
interface NotificationPreferences {
  emailDigest: "instant" | "daily" | "weekly" | "off";
  pushNotifications: boolean;
  categories: {
    comments: boolean;
    mentions: boolean;
    systemUpdates: boolean;
    marketing: boolean;
  };
}

function useNotificationPreferences() {
  const { data: preferences } = useQuery({
    queryKey: ["notification-preferences"],
    queryFn: () => notificationsApi.getPreferences(),
  });

  const updateMutation = useMutation({
    mutationFn: (updates: Partial<NotificationPreferences>) =>
      notificationsApi.updatePreferences(updates),
    onSuccess: (updated) => {
      queryClient.setQueryData(["notification-preferences"], updated);
    },
  });

  return { preferences, updatePreferences: updateMutation.mutate };
}

// The notification dispatcher (server-side, but the client should be
// AWARE of this contract) respects these preferences — e.g., a user
// who has disabled "comments" notifications should never receive a
// toast OR a notification center entry for new comments, regardless
// of how the notification was triggered server-side.
```

---

## Architecture Checklist

```
☐ Toast triggering API (useToast) is accessible from any component without
  prop drilling
☐ Toast stack is bounded (maxToasts) to prevent unbounded growth
☐ Notification center state is server-synced (read/unread persists
  across sessions and devices)
☐ Optimistic updates for mark-as-read with proper rollback
☐ Priority determines BOTH visual treatment AND whether something
  interrupts via toast vs. accumulates silently in the center
☐ Real-time delivery integrated cleanly with the existing WebSocket
  connection manager (not a separate, redundant connection)
☐ User-controllable notification preferences respected
```

## Accessibility Checklist

```
☐ Toast container uses aria-live (polite for routine, alert role for errors)
☐ Notification center bell icon has an accessible label reflecting unread count
☐ Keyboard navigable: dropdown can be opened/closed and navigated via keyboard
☐ Sufficient color contrast for all priority-level color coding
```

---

## Extension Ideas

```
- Notification grouping ("5 people liked your post" instead of 5 separate entries)
- Snooze/remind-later functionality
- Rich notifications with images/avatars
- Sound effects for high-priority notifications (with a mute preference)
- Browser push notifications (Notification API) when the tab isn't focused
- Notification analytics (open rate, click-through rate per notification type)
```

---

## 🔗 Related Topics

- [`projects/01-realtime-chat-application.md`](./01-realtime-chat-application.md) — WebSocket connection manager reused here
- [`animations/04-micro-interactions.md`](../animations/04-micro-interactions.md) — Toast animation patterns
- [`anti-patterns/01-prop-drilling.md`](../anti-patterns/01-prop-drilling.md) — Why Context is the right tool here

---

<div align="center">

**Next:** [`projects/10-file-upload-system.md`](./10-file-upload-system.md) →

</div>
