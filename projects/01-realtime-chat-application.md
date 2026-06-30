# 01 — Project: Real-Time Chat Application

> **"Projects differ from challenges in scope: a challenge builds one mechanism deeply, a project assembles many mechanisms into something a user could actually open and use. The skill being practiced here is integration — knowing how state management, networking, performance, and UX considerations interact when none of them can be designed in isolation."**

This project guide walks through building a production-quality real-time chat application: message lists, typing indicators, presence, optimistic sending, reconnection handling, and notification badges. It's structured as a build order with the key architectural decisions explained at each step, not a line-by-line tutorial — the goal is to practice making the same decisions a senior engineer makes when scoping this from a blank repository.

---

## 📚 What You'll Build

A chat application with: a conversation list sidebar, a message thread view, real-time message delivery via WebSocket, typing indicators, online presence, optimistic message sending with retry, unread badges, and reconnection handling after network drops.

---

## Requirements

```
FUNCTIONAL:
  - List of conversations, sorted by most recent activity
  - Message thread: paginated history + real-time new messages
  - Send messages with optimistic UI (appears instantly, confirms async)
  - Typing indicator ("Alice is typing...")
  - Online/offline presence per user
  - Unread message badges per conversation
  - Reconnect gracefully after network interruption

NON-FUNCTIONAL:
  - Message list must handle 10,000+ messages without performance degradation
  - UI must not "jump" when new messages arrive while scrolled up reading history
  - Must work correctly with React StrictMode's double-invocation in development
  - Should feel instant for the sender, even on slow connections
```

---

## Architecture Overview

```
COMPONENT TREE:
  <ChatApp>
    <ConversationList>           (sidebar: list of conversations + unread badges)
      <ConversationItem />        (per conversation: avatar, last message, unread count)
    <ChatThread>                  (main panel: selected conversation)
      <MessageList>                (virtualized list of messages)
        <MessageBubble />           (individual message, own vs other styling)
      <TypingIndicator />          (shows when someone is typing)
      <MessageComposer />          (text input + send button)

STATE LAYERS:
  1. Connection state: WebSocket status (connecting, open, closed, reconnecting)
  2. Conversation state: list of conversations, unread counts, last message preview
  3. Message state: per-conversation message history (paginated, cached)
  4. Presence state: which users are currently online
  5. Composer state: draft text, typing status (local to MessageComposer)
  6. Optimistic state: pending messages not yet confirmed by server

DATA FLOW:
  WebSocket → message dispatcher → updates conversation state + message state
  REST API → initial load of conversations + paginated message history
  User input → optimistic update → WebSocket send → server ack → reconcile
```

---

## Step 1 — WebSocket Connection Manager

The foundation: a connection manager that handles connecting, reconnecting with backoff, and message dispatching, decoupled from any specific component.

```typescript
type ConnectionStatus = "connecting" | "open" | "closed" | "reconnecting";

class ChatConnection {
  #ws: WebSocket | null = null;
  #status: ConnectionStatus = "closed";
  #reconnectAttempt = 0;
  #listeners = new Set<(msg: ServerMessage) => void>();
  #statusListeners = new Set<(status: ConnectionStatus) => void>();
  #messageQueue: ClientMessage[] = []; // queued while disconnected

  connect(url: string, token: string) {
    this.#setStatus(this.#reconnectAttempt > 0 ? "reconnecting" : "connecting");
    this.#ws = new WebSocket(`${url}?token=${token}`);

    this.#ws.onopen = () => {
      this.#reconnectAttempt = 0;
      this.#setStatus("open");
      this.#flushQueue();
    };

    this.#ws.onmessage = (e) => {
      const message = JSON.parse(e.data) as ServerMessage;
      this.#listeners.forEach((listener) => listener(message));
    };

    this.#ws.onclose = () => {
      this.#setStatus("closed");
      this.#scheduleReconnect(url, token);
    };

    this.#ws.onerror = () => {
      this.#ws?.close(); // triggers onclose → reconnect logic
    };
  }

  #scheduleReconnect(url: string, token: string) {
    const delay = Math.min(1000 * 2 ** this.#reconnectAttempt, 30_000);
    this.#reconnectAttempt++;
    setTimeout(() => this.connect(url, token), delay);
  }

  send(message: ClientMessage) {
    if (this.#status === "open" && this.#ws) {
      this.#ws.send(JSON.stringify(message));
    } else {
      this.#messageQueue.push(message); // queue until reconnected
    }
  }

  #flushQueue() {
    while (this.#messageQueue.length > 0) {
      const msg = this.#messageQueue.shift()!;
      this.#ws!.send(JSON.stringify(msg));
    }
  }

  onMessage(listener: (msg: ServerMessage) => void) {
    this.#listeners.add(listener);
    return () => this.#listeners.delete(listener);
  }

  onStatusChange(listener: (status: ConnectionStatus) => void) {
    this.#statusListeners.add(listener);
    return () => this.#statusListeners.delete(listener);
  }

  #setStatus(status: ConnectionStatus) {
    this.#status = status;
    this.#statusListeners.forEach((l) => l(status));
  }

  disconnect() {
    this.#ws?.close();
    this.#ws = null;
  }
}

export const chatConnection = new ChatConnection();
```

**Key decision:** the connection manager is a plain class, not a React hook or component. Connections need to persist across component remounts (especially important with React StrictMode's intentional double-mount in development) and shouldn't be tied to any specific component's lifecycle. React components subscribe TO this manager via hooks, rather than owning the connection themselves.

---

## Step 2 — React Bindings for the Connection

```typescript
function useConnectionStatus() {
  const [status, setStatus] = useState<ConnectionStatus>("closed");

  useEffect(() => {
    return chatConnection.onStatusChange(setStatus);
  }, []);

  return status;
}

function useChatMessages(conversationId: string) {
  const queryClient = useQueryClient();

  useEffect(() => {
    return chatConnection.onMessage((message) => {
      if (
        message.type === "new_message" &&
        message.conversationId === conversationId
      ) {
        // Append to the cached message list for this conversation
        queryClient.setQueryData<Message[]>(
          ["messages", conversationId],
          (prev = []) => [...prev, message.payload],
        );
      }
    });
  }, [conversationId, queryClient]);

  // Initial + paginated history via TanStack Query
  return useInfiniteQuery({
    queryKey: ["messages", conversationId],
    queryFn: ({ pageParam }) =>
      messagesApi.list(conversationId, { before: pageParam }),
    getNextPageParam: (lastPage) => lastPage.oldestMessageId,
  });
}
```

**Key decision:** real-time updates and paginated history both write into the SAME TanStack Query cache key. This means a new WebSocket message and a paginated "load more" both update the same source of truth, and the UI never needs to reconcile two separate data sources for the same conversation.

---

## Step 3 — Optimistic Message Sending

```typescript
function useSendMessage(conversationId: string) {
  const queryClient = useQueryClient();

  return useCallback(
    (text: string) => {
      const optimisticMessage: Message = {
        id: `pending-${crypto.randomUUID()}`, // temporary client-side ID
        conversationId,
        text,
        senderId: currentUserId,
        status: "sending",
        createdAt: new Date().toISOString(),
      };

      // 1. Apply optimistically — appears instantly
      queryClient.setQueryData<Message[]>(
        ["messages", conversationId],
        (prev = []) => [...prev, optimisticMessage],
      );

      // 2. Send via WebSocket
      chatConnection.send({
        type: "send_message",
        tempId: optimisticMessage.id,
        conversationId,
        text,
      });

      // 3. Server will respond with either:
      //    - message_sent: { tempId, realId } → reconcile the temp message
      //    - message_failed: { tempId, error } → mark as failed, allow retry
    },
    [conversationId, queryClient],
  );
}

// Reconciliation: listen for server confirmation
function useMessageReconciliation() {
  const queryClient = useQueryClient();

  useEffect(() => {
    return chatConnection.onMessage((message) => {
      if (message.type === "message_sent") {
        queryClient.setQueryData<Message[]>(
          ["messages", message.conversationId],
          (prev = []) =>
            prev.map((m) =>
              m.id === message.tempId
                ? { ...m, id: message.realId, status: "sent" }
                : m,
            ),
        );
      }

      if (message.type === "message_failed") {
        queryClient.setQueryData<Message[]>(
          ["messages", message.conversationId],
          (prev = []) =>
            prev.map((m) =>
              m.id === message.tempId ? { ...m, status: "failed" } : m,
            ),
        );
      }
    });
  }, [queryClient]);
}
```

**Key decision:** optimistic messages get a temporary client-generated ID (`pending-${uuid}`) so they can be rendered immediately and later matched against the server's confirmation via `tempId`. The `status` field (`sending` / `sent` / `failed`) drives the UI — a subtle opacity or spinner for pending messages, a retry button for failed ones.

---

## Step 4 — Virtualized Message List with "Stick to Bottom" Behavior

```jsx
function MessageList({ messages, hasMore, onLoadMore }) {
  const containerRef = useRef(null);
  const [isAtBottom, setIsAtBottom] = useState(true);
  const previousMessageCount = useRef(messages.length);

  // Detect if the user is scrolled near the bottom
  function handleScroll() {
    const el = containerRef.current;
    const threshold = 100; // px from bottom counts as "at bottom"
    const atBottom =
      el.scrollHeight - el.scrollTop - el.clientHeight < threshold;
    setIsAtBottom(atBottom);
  }

  // Auto-scroll to bottom ONLY if the user was already at the bottom
  // (don't yank the scroll position if they're reading history)
  useLayoutEffect(() => {
    const newMessageArrived = messages.length > previousMessageCount.current;
    previousMessageCount.current = messages.length;

    if (newMessageArrived && isAtBottom) {
      containerRef.current.scrollTop = containerRef.current.scrollHeight;
    }
  }, [messages.length, isAtBottom]);

  // Load more history when scrolled to top
  function handleScrollTop() {
    if (containerRef.current.scrollTop < 50 && hasMore) {
      const prevHeight = containerRef.current.scrollHeight;
      onLoadMore().then(() => {
        // Preserve scroll position after prepending older messages
        // (otherwise the view jumps because new content was added ABOVE)
        requestAnimationFrame(() => {
          containerRef.current.scrollTop =
            containerRef.current.scrollHeight - prevHeight;
        });
      });
    }
  }

  return (
    <div
      ref={containerRef}
      onScroll={() => {
        handleScroll();
        handleScrollTop();
      }}
      style={{ overflowY: "auto", height: "100%" }}
    >
      {messages.map((msg) => (
        <MessageBubble key={msg.id} message={msg} />
      ))}
    </div>
  );
}
```

**Key decision:** the "stick to bottom" behavior — only auto-scroll if the user was ALREADY at the bottom — is the single most important UX detail in a chat UI. Without this check, a user reading old messages would get yanked to the bottom every time a new message arrives, which is jarring and makes reading history impossible. For 10,000+ messages, combine this with the virtualization technique from [`challenges/01-build-a-virtualized-list.md`](../challenges/01-build-a-virtualized-list.md).

---

## Step 5 — Typing Indicators (Debounced + Auto-Expiring)

```typescript
function useTypingIndicator(conversationId: string) {
  const [typingUsers, setTypingUsers] = useState<Set<string>>(new Set());
  const expiryTimers = useRef<Map<string, ReturnType<typeof setTimeout>>>(
    new Map(),
  );

  useEffect(() => {
    return chatConnection.onMessage((message) => {
      if (
        message.type === "typing" &&
        message.conversationId === conversationId
      ) {
        setTypingUsers((prev) => new Set(prev).add(message.userId));

        // Auto-expire after 3s of no further typing events
        // (in case a "stopped typing" event never arrives, e.g., tab closed)
        clearTimeout(expiryTimers.current.get(message.userId));
        const timer = setTimeout(() => {
          setTypingUsers((prev) => {
            const next = new Set(prev);
            next.delete(message.userId);
            return next;
          });
        }, 3000);
        expiryTimers.current.set(message.userId, timer);
      }

      if (
        message.type === "stopped_typing" &&
        message.conversationId === conversationId
      ) {
        clearTimeout(expiryTimers.current.get(message.userId));
        setTypingUsers((prev) => {
          const next = new Set(prev);
          next.delete(message.userId);
          return next;
        });
      }
    });
  }, [conversationId]);

  return typingUsers;
}

// Sending side: debounce typing events to avoid flooding the server
function useTypingBroadcast(conversationId: string) {
  const isTypingRef = useRef(false);

  const broadcastTyping = useMemo(
    () =>
      debounce(() => {
        isTypingRef.current = false;
        chatConnection.send({ type: "stopped_typing", conversationId });
      }, 2000),
    [conversationId],
  );

  function handleInput() {
    if (!isTypingRef.current) {
      isTypingRef.current = true;
      chatConnection.send({ type: "typing", conversationId });
    }
    broadcastTyping(); // resets the "stopped typing" timer on each keystroke
  }

  return handleInput;
}
```

**Key decision:** typing indicators use TWO separate timeout mechanisms — one on the SENDER side (debounce: only fire "stopped typing" after 2s of no keystrokes) and one on the RECEIVER side (auto-expire after 3s, in case the "stopped typing" event never arrives due to a closed tab or dropped connection). Relying only on explicit "stopped typing" events would leave stale "Alice is typing..." indicators forever if Alice's tab crashes mid-message.

---

## Step 6 — Unread Badges and Notification Logic

```typescript
function useUnreadCounts() {
  const queryClient = useQueryClient();
  const [unreadCounts, setUnreadCounts] = useState<Map<string, number>>(
    new Map(),
  );
  const activeConversationId = useActiveConversationId(); // currently viewed conversation

  useEffect(() => {
    return chatConnection.onMessage((message) => {
      if (message.type !== "new_message") return;

      // Don't increment unread count for the conversation the user is
      // CURRENTLY viewing — they're already seeing it
      if (message.conversationId === activeConversationId) return;

      setUnreadCounts((prev) => {
        const next = new Map(prev);
        next.set(
          message.conversationId,
          (next.get(message.conversationId) ?? 0) + 1,
        );
        return next;
      });

      // Optional: browser notification if tab isn't focused
      if (document.hidden && Notification.permission === "granted") {
        new Notification(`New message from ${message.payload.senderName}`, {
          body: message.payload.text,
        });
      }
    });
  }, [activeConversationId]);

  function markAsRead(conversationId: string) {
    setUnreadCounts((prev) => {
      const next = new Map(prev);
      next.delete(conversationId);
      return next;
    });
    chatConnection.send({ type: "mark_read", conversationId });
  }

  return { unreadCounts, markAsRead };
}
```

---

## Performance and Reliability Checklist

```
☐ Message list virtualized for 10,000+ message conversations
☐ "Stick to bottom" only when user was already at bottom
☐ Optimistic messages have a distinguishable visual state (pending/sent/failed)
☐ WebSocket reconnects with exponential backoff, not immediate retry storms
☐ Messages sent while disconnected are queued and flushed on reconnect
☐ Typing indicators auto-expire (don't rely solely on explicit stop events)
☐ Unread counts don't increment for the currently-active conversation
☐ React StrictMode double-invocation doesn't create duplicate WebSocket connections
   (connection manager is a singleton outside React's render cycle)
☐ Scroll position preserved when loading older message history (prepend, not append)
☐ Browser notifications respect document.hidden and Notification.permission
```

---

## Extension Ideas

```
- Message reactions (emoji) with optimistic toggle
- Read receipts (double checkmark pattern)
- Message editing and deletion with "edited" indicator
- File/image attachments with upload progress
- @mentions with autocomplete
- Message search across conversation history
- End-to-end encryption (would require significant architecture changes)
- Voice/video call integration (WebRTC)
```

---

## 🔗 Related Topics

- [`networking/01-http-protocols.md`](../networking/01-http-protocols.md) — WebSocket protocol details
- [`challenges/01-build-a-virtualized-list.md`](../challenges/01-build-a-virtualized-list.md) — Virtualization for the message list
- [`anti-patterns/05-memory-leaks.md`](../anti-patterns/05-memory-leaks.md) — WebSocket cleanup patterns
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md) — Hook design used throughout

---

<div align="center">

**Next:** [`projects/02-ecommerce-product-page.md`](./02-ecommerce-product-page.md) →

</div>
