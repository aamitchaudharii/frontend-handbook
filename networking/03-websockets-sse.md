# 03 — WebSockets and Server-Sent Events

> **"HTTP is a conversation that starts and ends. WebSockets are a conversation that keeps going. Server-Sent Events are a broadcast you can tune into. Knowing which to use is knowing the difference between a dialogue, a monologue, and a telephone call."**

HTTP was designed for request-response: the client asks, the server answers, the connection closes. Real-time applications need more: live notifications, collaborative editing, live sports scores, trading dashboards, multiplayer games. WebSockets enable full-duplex persistent connections. Server-Sent Events (SSE) enable unidirectional server-to-client streaming with built-in reconnection. This document covers both technologies, their protocols, client and server implementations, reliability patterns, and how to choose between them.

---

## 📚 Table of Contents

1. [The Real-Time Problem](#1-the-real-time-problem)
2. [WebSocket Protocol](#2-websocket-protocol)
3. [WebSocket Client API](#3-websocket-client-api)
4. [WebSocket Reliability Patterns](#4-websocket-reliability-patterns)
5. [WebSocket Message Protocols](#5-websocket-message-protocols)
6. [Server-Sent Events (SSE)](#6-server-sent-events-sse)
7. [SSE vs WebSocket — The Decision](#7-sse-vs-websocket--the-decision)
8. [WebSocket in React](#8-websocket-in-react)
9. [WebSocket Authentication](#9-websocket-authentication)
10. [Binary WebSocket Messages](#10-binary-websocket-messages)
11. [Scaling WebSockets](#11-scaling-websockets)
12. [Good Practices](#12-good-practices)
13. [Bad Practices](#13-bad-practices)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview-Level Explanation](#15-interview-level-explanation)
16. [Exercises](#16-exercises)

---

## 1. The Real-Time Problem

### Why HTTP Request-Response Isn't Enough

```
POLLING (naive approach):
  Client: "anything new?" → Server: "no"  (100ms)
  Client: "anything new?" → Server: "no"  (100ms)
  Client: "anything new?" → Server: "YES" (100ms)

  Cost: N × (100ms RTT + server processing) per second
  Latency: up to poll interval (e.g., 1 second between events)
  Server load: high (many requests even when nothing changes)

LONG POLLING (better, but ugly):
  Client → server: "wait for something new"
  Server: holds request open until data available
  Server: responds when data arrives
  Client: immediately sends next long-poll request

  Better latency, but: complex server, 1 concurrent connection per client

WEBSOCKET (right tool):
  Client ←→ Server: persistent bidirectional connection
  Server pushes immediately when data changes
  Zero polling cost

SERVER-SENT EVENTS (unidirectional):
  Client ← Server: persistent one-way stream
  Browser maintains connection, receives events as they arrive
  Built-in reconnection, works over HTTP
```

---

## 2. WebSocket Protocol

### The Handshake

```
HTTP Upgrade Handshake:

Client → Server:
  GET /ws HTTP/1.1
  Host: api.example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==  ← random base64
  Sec-WebSocket-Version: 13
  Sec-WebSocket-Protocol: chat, json-rpc  ← optional subprotocols

Server → Client:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=  ← HMAC of client key
  Sec-WebSocket-Protocol: chat  ← selected subprotocol

After 101: connection upgraded — no more HTTP, just WebSocket frames
```

### WebSocket Frame Structure

```
WEBSOCKET FRAME:

  FIN|RSV1|RSV2|RSV3 (4 bits) | Opcode (4 bits)
  MASK bit | Payload length (7 bits)
  [Extended payload length: 2 or 8 bytes if needed]
  [Masking key: 4 bytes if MASK=1]
  [Payload data]

OPCODES:
  0x0: Continuation frame
  0x1: Text frame (UTF-8 encoded)
  0x2: Binary frame
  0x8: Close
  0x9: Ping
  0xA: Pong

CLIENT → SERVER: must be masked (XOR with masking key)
SERVER → CLIENT: not masked

Masking reason: protects against proxy cache poisoning attacks
```

---

## 3. WebSocket Client API

### Basic Connection

```javascript
// Create WebSocket connection
const ws = new WebSocket(
  "wss://api.example.com/ws", // wss = WebSocket Secure (TLS)
  ["json-rpc", "chat"], // optional subprotocols (server picks one)
);

// Connection opened
ws.addEventListener("open", (event) => {
  console.log("WebSocket connected");
  console.log("Selected subprotocol:", ws.protocol); // 'json-rpc' or 'chat'
  console.log("Extensions:", ws.extensions);

  // Send initial message
  ws.send(JSON.stringify({ type: "SUBSCRIBE", channel: "prices" }));
});

// Message received
ws.addEventListener("message", (event) => {
  console.log("Message:", event.data); // string or Blob or ArrayBuffer
  const message = JSON.parse(event.data);
  handleMessage(message);
});

// Error occurred
ws.addEventListener("error", (event) => {
  // Note: error events contain NO useful information about what went wrong
  // The subsequent 'close' event has the code and reason
  console.error("WebSocket error");
});

// Connection closed
ws.addEventListener("close", (event) => {
  console.log(`Closed: code=${event.code}, reason=${event.reason}`);
  console.log(`Was clean: ${event.wasClean}`);
  // Common close codes:
  // 1000: Normal closure
  // 1001: Going away (tab closed, server restart)
  // 1006: Abnormal closure (no close frame — network drop)
  // 1008: Policy violation (auth failure)
  // 1011: Internal server error
  // 4000-4999: Application-defined codes
});
```

### WebSocket States and Buffering

```javascript
// readyState values
ws.readyState; // 0=CONNECTING, 1=OPEN, 2=CLOSING, 3=CLOSED

// Sending messages
ws.send("string message");
ws.send(JSON.stringify({ type: "ping" }));
ws.send(new ArrayBuffer(16)); // binary
ws.send(new Blob([data])); // blob

// Buffered amount (bytes queued but not yet sent)
ws.bufferedAmount; // 0 = all sent, > 0 = backpressure
// Check before sending to avoid overwhelming slow connections
if (ws.bufferedAmount < 16_384) {
  // 16KB threshold
  ws.send(data);
} else {
  // Skip or queue
}

// Closing
ws.close(1000, "Normal closure"); // graceful close
ws.close(1008, "Authentication error"); // policy violation
```

---

## 4. WebSocket Reliability Patterns

### Automatic Reconnection

```typescript
class ReliableWebSocket {
  #url: string;
  #protocols: string[];
  #ws: WebSocket | null = null;
  #messageHandlers: Set<(data: unknown) => void> = new Set();
  #closeHandlers: Set<(event: CloseEvent) => void> = new Set();
  #openHandlers: Set<() => void> = new Set();

  #reconnectAttempts = 0;
  #maxReconnects = 10;
  #reconnecting = false;
  #shouldReconnect = true;

  // Exponential backoff: 1s, 2s, 4s, 8s, 16s, 30s (max)
  #getDelay(): number {
    return Math.min(1000 * 2 ** this.#reconnectAttempts, 30_000);
  }

  constructor(url: string, protocols?: string | string[]) {
    this.#url = url;
    this.#protocols = Array.isArray(protocols)
      ? protocols
      : protocols
        ? [protocols]
        : [];
    this.#connect();
  }

  #connect(): void {
    this.#ws = new WebSocket(this.#url, this.#protocols);

    this.#ws.addEventListener("open", () => {
      this.#reconnectAttempts = 0; // reset on successful connection
      this.#reconnecting = false;
      this.#openHandlers.forEach((fn) => fn());
    });

    this.#ws.addEventListener("message", (event) => {
      try {
        const data = JSON.parse(event.data);
        this.#messageHandlers.forEach((fn) => fn(data));
      } catch {
        this.#messageHandlers.forEach((fn) => fn(event.data));
      }
    });

    this.#ws.addEventListener("close", (event) => {
      this.#closeHandlers.forEach((fn) => fn(event));

      if (
        this.#shouldReconnect &&
        event.code !== 1000 && // not normal closure
        event.code !== 1008 && // not auth failure
        this.#reconnectAttempts < this.#maxReconnects
      ) {
        this.#scheduleReconnect();
      }
    });
  }

  #scheduleReconnect(): void {
    if (this.#reconnecting) return;
    this.#reconnecting = true;

    const delay = this.#getDelay();
    this.#reconnectAttempts++;

    console.log(
      `[WS] Reconnecting in ${delay}ms (attempt ${this.#reconnectAttempts})`,
    );
    setTimeout(() => this.#connect(), delay);
  }

  send(data: unknown): void {
    if (this.#ws?.readyState !== WebSocket.OPEN) {
      throw new Error("WebSocket is not connected");
    }
    this.#ws.send(typeof data === "string" ? data : JSON.stringify(data));
  }

  // Safe send: queues messages while reconnecting
  safeSend(data: unknown): Promise<void> {
    return new Promise((resolve, reject) => {
      if (this.#ws?.readyState === WebSocket.OPEN) {
        this.send(data);
        resolve();
      } else {
        // Wait for reconnection
        const handler = () => {
          this.send(data);
          this.#openHandlers.delete(handler);
          resolve();
        };
        this.#openHandlers.add(handler);
        // Timeout after 10 seconds
        setTimeout(() => {
          this.#openHandlers.delete(handler);
          reject(new Error("WebSocket reconnection timeout"));
        }, 10_000);
      }
    });
  }

  on(event: "message", handler: (data: unknown) => void): () => void;
  on(event: "open", handler: () => void): () => void;
  on(event: "close", handler: (event: CloseEvent) => void): () => void;
  on(event: string, handler: Function): () => void {
    if (event === "message") {
      this.#messageHandlers.add(handler as (data: unknown) => void);
      return () =>
        this.#messageHandlers.delete(handler as (data: unknown) => void);
    }
    if (event === "open") {
      this.#openHandlers.add(handler as () => void);
      return () => this.#openHandlers.delete(handler as () => void);
    }
    if (event === "close") {
      this.#closeHandlers.add(handler as (event: CloseEvent) => void);
      return () =>
        this.#closeHandlers.delete(handler as (event: CloseEvent) => void);
    }
    return () => {};
  }

  disconnect(): void {
    this.#shouldReconnect = false;
    this.#ws?.close(1000, "Normal closure");
  }
}
```

### Heartbeat / Ping-Pong

```typescript
class HeartbeatWebSocket extends ReliableWebSocket {
  #pingInterval: ReturnType<typeof setInterval> | null = null;
  #pongTimeout: ReturnType<typeof setTimeout> | null = null;
  readonly #pingIntervalMs: number;
  readonly #pongTimeoutMs: number;

  constructor(
    url: string,
    { pingInterval = 30_000, pongTimeout = 5_000 } = {},
  ) {
    super(url);
    this.#pingIntervalMs = pingInterval;
    this.#pongTimeoutMs = pongTimeout;

    this.on("open", () => this.#startHeartbeat());
    this.on("close", () => this.#stopHeartbeat());
    this.on("message", (data: unknown) => {
      if ((data as { type?: string }).type === "pong") {
        this.#onPong();
      }
    });
  }

  #startHeartbeat(): void {
    this.#pingInterval = setInterval(
      () => this.#sendPing(),
      this.#pingIntervalMs,
    );
  }

  #stopHeartbeat(): void {
    if (this.#pingInterval) clearInterval(this.#pingInterval);
    if (this.#pongTimeout) clearTimeout(this.#pongTimeout);
  }

  #sendPing(): void {
    this.send({ type: "ping", timestamp: Date.now() });

    // If no pong within timeout: connection is dead
    this.#pongTimeout = setTimeout(() => {
      console.warn("[WS] Pong timeout — forcing reconnect");
      // WebSocket spec: send ping frame (0x9), server auto-replies with pong (0xA)
      // Or: application-level ping/pong if server doesn't implement WS ping
    }, this.#pongTimeoutMs);
  }

  #onPong(): void {
    if (this.#pongTimeout) {
      clearTimeout(this.#pongTimeout);
      this.#pongTimeout = null;
    }
  }
}
```

---

## 5. WebSocket Message Protocols

### JSON Envelope Pattern

```typescript
// Define a typed message protocol
type ClientMessage =
  | { type: "SUBSCRIBE"; channel: string }
  | { type: "UNSUBSCRIBE"; channel: string }
  | { type: "SEND"; channel: string; content: string }
  | { type: "PING"; timestamp: number };

type ServerMessage =
  | {
      type: "MESSAGE";
      channel: string;
      content: string;
      sender: string;
      timestamp: number;
    }
  | { type: "SUBSCRIBED"; channel: string; memberCount: number }
  | { type: "ERROR"; code: number; message: string }
  | { type: "PONG"; timestamp: number };

class TypedWebSocket {
  #ws: ReliableWebSocket;

  constructor(url: string) {
    this.#ws = new ReliableWebSocket(url);
  }

  send(message: ClientMessage): void {
    this.#ws.send(message);
  }

  on<T extends ServerMessage["type"]>(
    type: T,
    handler: (message: Extract<ServerMessage, { type: T }>) => void,
  ): () => void {
    return this.#ws.on("message", (data: unknown) => {
      const message = data as ServerMessage;
      if (message.type === type) {
        handler(message as Extract<ServerMessage, { type: T }>);
      }
    });
  }
}

// Usage:
const chat = new TypedWebSocket("wss://api.example.com/chat");

chat.send({ type: "SUBSCRIBE", channel: "general" });

const unsubscribe = chat.on("MESSAGE", (msg) => {
  console.log(`[${msg.channel}] ${msg.sender}: ${msg.content}`);
});

// Cleanup
unsubscribe();
```

### Request-Response over WebSocket

```typescript
// WebSocket RPC: send request, await response
class WebSocketRPC {
  #ws: ReliableWebSocket;
  #pending: Map<
    string,
    {
      resolve: Function;
      reject: Function;
      timeout: ReturnType<typeof setTimeout>;
    }
  > = new Map();
  #nextId = 0;

  constructor(url: string) {
    this.#ws = new ReliableWebSocket(url);
    this.#ws.on("message", (data: unknown) => this.#handleResponse(data));
  }

  call<T>(method: string, params?: unknown, timeoutMs = 10_000): Promise<T> {
    return new Promise((resolve, reject) => {
      const id = String(++this.#nextId);
      const timeout = setTimeout(() => {
        this.#pending.delete(id);
        reject(new Error(`RPC timeout: ${method} (${timeoutMs}ms)`));
      }, timeoutMs);

      this.#pending.set(id, { resolve, reject, timeout });

      this.#ws.send({
        jsonrpc: "2.0",
        id,
        method,
        params,
      });
    });
  }

  #handleResponse(data: unknown): void {
    const response = data as {
      id?: string;
      result?: unknown;
      error?: { code: number; message: string };
    };
    if (!response.id) return; // notification, not a response

    const pending = this.#pending.get(response.id);
    if (!pending) return;

    clearTimeout(pending.timeout);
    this.#pending.delete(response.id);

    if (response.error) {
      pending.reject(
        new Error(
          `RPC error ${response.error.code}: ${response.error.message}`,
        ),
      );
    } else {
      pending.resolve(response.result);
    }
  }
}

// Usage:
const rpc = new WebSocketRPC("wss://api.example.com/rpc");
const user = await rpc.call<User>("getUser", { id: "42" });
const result = await rpc.call<Order>("placeOrder", { items: cart });
```

---

## 6. Server-Sent Events (SSE)

SSE is a one-way server-to-client streaming protocol built on HTTP.

### SSE Protocol Format

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Transfer-Encoding: chunked

data: {"type":"price","symbol":"AAPL","price":178.23}\n\n

event: order-update\n
data: {"orderId":"42","status":"shipped"}\n\n

id: 1234\n
data: {"type":"notification","message":"Your order shipped!"}\n\n

: this is a comment (ignored by client)\n\n

retry: 5000\n
data: {"type":"reconnect-hint"}\n\n
```

### EventSource API

```javascript
// Create SSE connection
const source = new EventSource("/api/events", {
  withCredentials: true, // send cookies
});

// Default 'message' events (no event: field in stream)
source.addEventListener("message", (event) => {
  const data = JSON.parse(event.data);
  console.log("Message:", data);
  console.log("Last event ID:", event.lastEventId); // for reconnection
});

// Named events (matching event: field in stream)
source.addEventListener("order-update", (event) => {
  const order = JSON.parse(event.data);
  updateOrderStatus(order);
});

source.addEventListener("price-update", (event) => {
  const { symbol, price } = JSON.parse(event.data);
  updatePriceDisplay(symbol, price);
});

// Connection opened
source.addEventListener("open", () => {
  console.log("SSE connected, state:", source.readyState);
  // 0 = CONNECTING, 1 = OPEN, 2 = CLOSED
});

// Error / reconnection
source.addEventListener("error", (event) => {
  if (source.readyState === EventSource.CLOSED) {
    console.error("SSE connection permanently closed");
  }
  // readyState === CONNECTING: browser is automatically reconnecting
  // Browser sends Last-Event-ID header so server can resume from last seen event
});

// Close
source.close(); // manual close — browser won't reconnect
```

### SSE Reconnection and Last-Event-ID

```javascript
// Server-side: send event IDs for resumption
function sendEvents(res, lastEventId) {
  const events = getEventsSince(lastEventId); // fetch missed events

  events.forEach((event) => {
    res.write(`id: ${event.id}\n`);
    res.write(`event: ${event.type}\n`);
    res.write(`data: ${JSON.stringify(event.payload)}\n\n`);
  });

  // Subscribe to new events and push as they arrive
}

// Client: browser automatically sends Last-Event-ID on reconnect
// GET /api/events
// Last-Event-ID: 1234  ← server picks up from event 1234

// Browser reconnection:
// - Automatic (built-in to EventSource)
// - Default delay: 3 seconds (customizable via retry: field)
// - Sends Last-Event-ID so server can resume the stream
// - Stops reconnecting if: server sends HTTP 204 or non-2xx
```

### Custom SSE with Fetch (for more control)

```javascript
// Fetch-based SSE: handles auth headers, custom retry logic
async function* listenToSSE(url, options = {}) {
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      Accept: "text/event-stream",
      "Cache-Control": "no-cache",
    },
  });

  if (!response.ok)
    throw new Error(`SSE connection failed: ${response.status}`);

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });

      // Parse SSE messages (separated by double newline)
      const messages = buffer.split("\n\n");
      buffer = messages.pop() ?? "";

      for (const message of messages) {
        const event = parseSSEMessage(message);
        if (event) yield event;
      }
    }
  } finally {
    reader.releaseLock();
  }
}

function parseSSEMessage(raw) {
  const lines = raw.split("\n");
  let data = "";
  let event = "message";
  let id = "";
  let retry = undefined;

  for (const line of lines) {
    if (line.startsWith("data: ")) data += (data ? "\n" : "") + line.slice(6);
    if (line.startsWith("event: ")) event = line.slice(7);
    if (line.startsWith("id: ")) id = line.slice(4);
    if (line.startsWith("retry: ")) retry = Number(line.slice(7));
  }

  return data ? { data, event, id, retry } : null;
}

// Usage with auth:
for await (const event of listenToSSE("/api/events", {
  headers: { Authorization: `Bearer ${token}` },
})) {
  handleEvent(event);
}
```

---

## 7. SSE vs WebSocket — The Decision

```
              SSE                    WebSocket
              ──────────────         ──────────────
Direction:    Server → Client        Bidirectional (both ways)
Protocol:     HTTP (port 80/443)     WS (ws://) or WSS (wss://)
Reconnect:    Built-in (automatic)   Must implement yourself
Backoff:      Built-in               Manual
Last-Event-ID: Built-in              Manual (with your own protocol)
Binary data:  No (text only)         Yes (binary frames)
Headers:      Works (standard HTTP)  Needs workaround (auth via token in URL)
Multiplexing: Over HTTP/2            One connection per stream (usually)
Proxies:      Works everywhere       Some proxies block WS
Complexity:   Low                    Medium
Max connections: 6 per origin (H1)   Many (separate from HTTP pool)
               Unlimited (H2/H3)

USE SSE FOR:
  ✓ Notifications (user, system, alerts)
  ✓ Live feeds (news, prices, social)
  ✓ Progress events (file processing, jobs)
  ✓ Server log tailing
  ✓ When you need auth headers (can't do WS easily)
  ✓ When proxy/firewall compatibility matters

USE WEBSOCKET FOR:
  ✓ Chat / messaging (bidirectional)
  ✓ Collaborative editing (both read and write)
  ✓ Gaming (low-latency, frequent bidirectional)
  ✓ Binary data (audio, video, sensor data)
  ✓ Custom protocols (RPC, pub/sub)
  ✓ When you need bidirectional communication
```

---

## 8. WebSocket in React

```typescript
// React hook: managed WebSocket lifecycle
function useWebSocket<T>(url: string | null) {
  const [socket, setSocket]   = useState<ReliableWebSocket | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const messageHandlers = useRef<Set<(data: T) => void>>(new Set());

  useEffect(() => {
    if (!url) return;

    const ws = new ReliableWebSocket(url);

    const unsubOpen  = ws.on('open',  () => setIsConnected(true));
    const unsubClose = ws.on('close', () => setIsConnected(false));
    const unsubMsg   = ws.on('message', (data: T) => {
      messageHandlers.current.forEach(fn => fn(data));
    });

    setSocket(ws);

    return () => {
      unsubOpen();
      unsubClose();
      unsubMsg();
      ws.disconnect();
      setSocket(null);
      setIsConnected(false);
    };
  }, [url]);

  const subscribe = useCallback((handler: (data: T) => void) => {
    messageHandlers.current.add(handler);
    return () => messageHandlers.current.delete(handler);
  }, []);

  const send = useCallback((data: unknown) => {
    socket?.send(data);
  }, [socket]);

  return { socket, isConnected, subscribe, send };
}

// Context: share one WebSocket connection across components
const WebSocketContext = createContext<ReturnType<typeof useWebSocket> | null>(null);

function WebSocketProvider({ url, children }: { url: string; children: ReactNode }) {
  const ws = useWebSocket(url);
  return <WebSocketContext.Provider value={ws}>{children}</WebSocketContext.Provider>;
}

function useWS() {
  const ctx = useContext(WebSocketContext);
  if (!ctx) throw new Error('useWS must be used within WebSocketProvider');
  return ctx;
}

// Component: subscribe to specific message types
function PriceDisplay({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);
  const { subscribe }     = useWS();

  useEffect(() => {
    return subscribe((message: { type: string; symbol: string; price: number }) => {
      if (message.type === 'PRICE' && message.symbol === symbol) {
        setPrice(message.price);
      }
    });
  }, [subscribe, symbol]);

  return <span>{price !== null ? `$${price.toFixed(2)}` : '...'}</span>;
}
```

---

## 9. WebSocket Authentication

WebSockets don't support custom headers after the initial HTTP upgrade. Use tokens:

```javascript
// Method 1: Token in URL query parameter
// (avoid if possible — tokens in URLs appear in logs, history, Referer headers)
const ws = new WebSocket(
  `wss://api.example.com/ws?token=${encodeURIComponent(accessToken)}`,
);

// Method 2: First message authentication (recommended)
const ws = new WebSocket("wss://api.example.com/ws");

ws.addEventListener("open", () => {
  // First message: authenticate
  ws.send(
    JSON.stringify({
      type: "AUTH",
      token: accessToken,
    }),
  );
});

ws.addEventListener("message", (event) => {
  const message = JSON.parse(event.data);
  if (message.type === "AUTH_SUCCESS") {
    // Now subscribed and receiving events
    ws.send(JSON.stringify({ type: "SUBSCRIBE", channel: "prices" }));
  }
  if (message.type === "AUTH_FAILURE") {
    ws.close(1008, "Authentication failed");
  }
});

// Method 3: Cookie-based (most transparent if same-origin)
// Works if both HTTP API and WebSocket server share a session cookie
const ws = new WebSocket("wss://api.example.com/ws");
// Browser automatically sends cookies if same origin
```

### Token Refresh for Long-Lived WebSocket Connections

```javascript
class AuthenticatedWebSocket {
  #ws: ReliableWebSocket;
  #getToken: () => Promise<string>;
  #tokenRefreshTimer: ReturnType<typeof setInterval> | null = null;

  constructor(url: string, getToken: () => Promise<string>) {
    this.#ws       = new ReliableWebSocket(url);
    this.#getToken = getToken;

    this.#ws.on('open', async () => {
      const token = await this.#getToken();
      this.#ws.send({ type: 'AUTH', token });
      this.#startTokenRefresh();
    });

    this.#ws.on('close', () => {
      if (this.#tokenRefreshTimer) clearInterval(this.#tokenRefreshTimer);
    });
  }

  #startTokenRefresh(): void {
    // Refresh token every 45 minutes (assuming 1 hour expiry)
    this.#tokenRefreshTimer = setInterval(async () => {
      const token = await this.#getToken(); // returns fresh token
      this.#ws.send({ type: 'REFRESH_TOKEN', token });
    }, 45 * 60_000);
  }
}
```

---

## 10. Binary WebSocket Messages

```javascript
// Send binary data
const ws = new WebSocket("wss://api.example.com/binary");
ws.binaryType = "arraybuffer"; // 'arraybuffer' | 'blob' (default: 'blob')

// Send typed array
const data = new Float32Array([1.0, 2.0, 3.0]);
ws.send(data.buffer); // send the underlying ArrayBuffer

// Receive binary
ws.addEventListener("message", (event) => {
  if (event.data instanceof ArrayBuffer) {
    const view = new DataView(event.data);
    const messageType = view.getUint8(0); // first byte: message type
    const timestamp = view.getFloat64(1); // next 8 bytes: timestamp
    const value = view.getFloat32(9); // next 4 bytes: value
    handleBinaryMessage(messageType, timestamp, value);
  }
});

// Binary protocol design: header + payload
function encodeSensorData(sensorId, readings) {
  const header = 13; // bytes: 1 type + 4 sensorId + 8 timestamp
  const payload = readings.length * 4; // float32 per reading
  const buffer = new ArrayBuffer(header + payload);
  const view = new DataView(buffer);

  view.setUint8(0, 0x01); // message type: sensor data
  view.setUint32(1, sensorId, true); // little-endian
  view.setFloat64(5, Date.now(), true); // timestamp
  readings.forEach((r, i) => {
    view.setFloat32(header + i * 4, r, true);
  });

  return buffer;
}

ws.send(encodeSensorData(42, [23.5, 24.1, 23.8, 24.0]));
```

---

## 11. Scaling WebSockets

WebSockets require stateful connections — scaling requires care:

```
CHALLENGE:
  User A connected to Server 1
  User B connected to Server 2
  A sends message to B → Server 1 needs to reach Server 2

SOLUTIONS:

1. STICKY SESSIONS (simple, limited):
   Load balancer routes each user to the same server
   Easy to implement
   Problem: server failure drops ALL connections routed to it

2. PUB/SUB MESSAGE BROKER (recommended):
   All servers subscribe to a shared broker (Redis, NATS, Kafka)
   Any server can publish; subscribers on ALL servers receive

   User A → Server 1 → publishes to Redis
   Redis → all servers → Server 2 delivers to User B

   Libraries: Socket.io with Redis adapter, Ably, Pusher

3. MANAGED WS SERVICE:
   Use Ably, Pusher, AWS AppSync, Cloudflare Durable Objects
   They handle scaling, reconnection, presence, history
   Trade: cost and vendor dependency
```

### Redis Pub/Sub Pattern (Backend)

```javascript
// Server (Node.js): bridge WebSocket to Redis pub/sub
const redis = require("redis");
const WebSocket = require("ws");

const publisher = redis.createClient();
const subscriber = redis.createClient();
const wss = new WebSocket.Server({ port: 8080 });

// Each WebSocket client subscribes to relevant channels
wss.on("connection", (ws, req) => {
  const userId = extractUserId(req); // from auth token
  const channels = [`user:${userId}`, "broadcasts"];

  // Subscribe to Redis channels for this user
  subscriber.subscribe(channels, (message, channel) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(message); // forward Redis message to WebSocket client
    }
  });

  ws.on("message", (data) => {
    const msg = JSON.parse(data);
    const target = msg.targetUserId ? `user:${msg.targetUserId}` : "broadcasts";

    // Publish to Redis — all servers will receive and forward to target user
    publisher.publish(
      target,
      JSON.stringify({
        ...msg,
        fromUserId: userId,
        timestamp: Date.now(),
      }),
    );
  });

  ws.on("close", () => subscriber.unsubscribe(channels));
});
```

---

## 12. Good Practices

### ✅ Always implement reconnection logic

```javascript
// WebSockets drop — always reconnect
// Use exponential backoff to avoid hammering the server
// See ReliableWebSocket implementation above
```

### ✅ Handle connection state in UI

```typescript
// ✅ Show connection status to users
function ConnectionIndicator() {
  const { isConnected } = useWS();
  return (
    <span className={`indicator ${isConnected ? 'green' : 'red'}`}>
      {isConnected ? 'Live' : 'Reconnecting...'}
    </span>
  );
}
```

### ✅ Use SSE over WebSocket for server-push-only scenarios

```javascript
// ✅ Notifications, live feeds — SSE is simpler
// Built-in reconnection, works with regular HTTP auth headers
const source = new EventSource("/api/notifications", { withCredentials: true });
```

### ✅ Clean up on unmount

```javascript
// ✅ Prevent memory leaks — close/unsubscribe when done
useEffect(() => {
  const ws = new WebSocket(url);
  return () => ws.close(1000); // cleanup on unmount
}, []);
```

---

## 13. Bad Practices

### ❌ Creating a new WebSocket connection per component

```javascript
// ❌ 50 components = 50 WebSocket connections
function PriceDisplay() {
  const [price, setPrice] = useState(null);
  useEffect(() => {
    const ws = new WebSocket("/ws/prices"); // new connection per component!
    ws.onmessage = (e) => setPrice(JSON.parse(e.data).price);
    return () => ws.close();
  }, []);
}

// ✅ Share one connection via context (see WebSocketProvider above)
```

### ❌ Ignoring close codes when deciding whether to reconnect

```javascript
// ❌ Always reconnecting — even on auth failure (1008) or server told us not to (4xxx)
ws.onclose = () => reconnect(); // blindly reconnects

// ✅ Only reconnect when appropriate
ws.onclose = (event) => {
  if (event.code === 1000) return; // normal close: don't reconnect
  if (event.code === 1008) return; // auth failure: need new token first
  if (event.code >= 4000 && event.code < 4100) return; // server says don't retry
  reconnect(); // all other cases: reconnect
};
```

### ❌ Sending auth tokens in WebSocket URL

```javascript
// ❌ Token visible in browser history, logs, Referer headers
const ws = new WebSocket(`wss://api.example.com/ws?auth=${secretToken}`);

// ✅ Send as first message after connection
ws.addEventListener("open", () => {
  ws.send(JSON.stringify({ type: "AUTH", token: secretToken }));
});
```

---

## 14. Common Mistakes

### Mistake 1 — Not handling backpressure

```javascript
// ❌ Sending faster than socket can handle
function sendHighFrequencyUpdates() {
  setInterval(() => {
    ws.send(JSON.stringify(getCurrentData())); // 100 msg/sec, socket can only handle 10
    // ws.bufferedAmount grows indefinitely → memory leak, eventual dropped connection
  }, 10);
}

// ✅ Check bufferedAmount before sending
function sendHighFrequencyUpdates() {
  setInterval(() => {
    if (ws.bufferedAmount < 16_384) {
      // only send if buffer is reasonably empty
      ws.send(JSON.stringify(getCurrentData()));
    }
    // else: skip this update (client will get next one)
  }, 10);
}
```

### Mistake 2 — SSE's 6-connection limit on HTTP/1.1

```javascript
// EventSource uses HTTP connections from the browser's connection pool
// HTTP/1.1: max 6 connections per origin
// Multiple EventSource instances to same origin can exhaust the pool

// ❌ Three SSE connections on HTTP/1.1 = 3 of 6 connections used
const news = new EventSource("/api/news-feed");
const prices = new EventSource("/api/price-feed");
const notifs = new EventSource("/api/notifications");

// ✅ Multiplex all SSE events on one connection with event types
const events = new EventSource("/api/events"); // single connection
events.addEventListener("news", handleNews);
events.addEventListener("prices", handlePrices);
events.addEventListener("notifs", handleNotifications);

// Or: use HTTP/2 (no 6-connection limit)
```

### Mistake 3 — Forgetting that WebSocket isn't HTTP

```javascript
// ❌ Assuming CORS applies to WebSocket connections
// WebSocket upgrade is NOT subject to CORS
// The server must validate the Origin header manually!

// Server-side validation:
wss.on("connection", (ws, request) => {
  const origin = request.headers["origin"];
  const allowedOrigins = [
    "https://app.example.com",
    "https://staging.example.com",
  ];

  if (!allowedOrigins.includes(origin)) {
    ws.close(1008, "Origin not allowed");
    return;
  }
  // ...
});
```

---

## 15. Interview-Level Explanation

> **"How do WebSockets work? When would you use SSE instead?"**

**Strong answer:**

> "WebSockets start as a regular HTTP request with an Upgrade header. If the server accepts, it responds with 101 Switching Protocols, and from that point the TCP connection is handed over to the WebSocket protocol. What was an HTTP connection is now a persistent, bidirectional, message-oriented channel. Either side can send frames at any time — the server can push data without waiting for a client request.
>
> The client API is event-based: you attach handlers to `open`, `message`, `error`, and `close` events. WebSocket itself doesn't specify what messages look like — you define a protocol on top. Most apps use JSON envelopes with a type field, some use binary protocols for performance-sensitive cases.
>
> Reliability is the key implementation challenge WebSocket doesn't solve for you. You have to implement reconnection yourself, with exponential backoff to avoid thundering herd when a server restarts. You also need to implement heartbeating — ping/pong — to detect zombie connections where the TCP socket appears alive but is actually dead. And you need to think about authentication, since WebSocket upgrades don't carry custom headers after the initial handshake — the common approach is to authenticate via the first message after connection.
>
> Server-Sent Events are the right choice when communication is only server-to-client. SSE is just HTTP with a specific content type and message format — the server sends a chunked response that never ends, with events separated by double newlines. The browser's EventSource API abstracts this and adds automatic reconnection with Last-Event-ID support so the server can resume the stream after a disconnect. Because it's regular HTTP, it works through any proxy, respects auth headers, and benefits from HTTP/2 multiplexing.
>
> The practical heuristic: use WebSocket for bidirectional communication — chat, collaborative editing, gaming. Use SSE for unidirectional server push — notifications, live feeds, progress events, log tailing. SSE is significantly simpler and more reliable in production; the built-in reconnection alone saves substantial effort."

---

## 16. Exercises

### Exercise 1 — Design a live collaboration system

You're building a collaborative document editor (like Google Docs) for up to 100 simultaneous editors on one document. Design:

1. The WebSocket message protocol
2. How you handle conflicts (two users edit the same paragraph)
3. How you handle disconnected users rejoining and catching up on missed changes

<details>
<summary>Solution</summary>

```typescript
// 1. MESSAGE PROTOCOL

type ClientMessage =
  | { type: "JOIN"; docId: string; lastSeenVersion: number }
  | { type: "OPERATION"; docId: string; op: Operation; version: number }
  | { type: "CURSOR"; docId: string; position: CursorPosition };

type ServerMessage =
  | {
      type: "JOINED";
      docId: string;
      content: string;
      version: number;
      peers: Peer[];
    }
  | { type: "OPERATION"; op: Operation; version: number; userId: string }
  | { type: "CURSOR"; userId: string; position: CursorPosition }
  | { type: "PEER_JOIN"; user: Peer }
  | { type: "PEER_LEAVE"; userId: string }
  | {
      type: "CATCH_UP";
      ops: Array<{ op: Operation; version: number; userId: string }>;
    };

type Operation =
  | { type: "INSERT"; at: number; text: string }
  | { type: "DELETE"; at: number; length: number }
  | { type: "RETAIN"; length: number };

// 2. CONFLICT RESOLUTION: Operational Transformation (OT)

function transformOperation(op1: Operation, op2: Operation): Operation {
  // If op1 is an INSERT at position P1, and op2 is an INSERT at position P2:
  // If P2 <= P1: op1's position shifts right by op2's length
  if (op1.type === "INSERT" && op2.type === "INSERT") {
    if (op2.at <= op1.at) {
      return { ...op1, at: op1.at + op2.text.length };
    }
  }
  // If op2 is a DELETE before op1's position: op1 shifts left
  if (op1.type === "INSERT" && op2.type === "DELETE") {
    if (op2.at < op1.at) {
      return { ...op1, at: Math.max(op2.at, op1.at - op2.length) };
    }
  }
  return op1; // no transformation needed
}

// 3. CATCH-UP ON REJOIN

// Server: stores operation history with version numbers
// When client joins with lastSeenVersion < current:
// Server sends CATCH_UP with all ops since lastSeenVersion

// Client handler:
ws.on("JOINED", ({ content, version, peers }) => {
  initDocument(content);
  currentVersion = version;
  showPeers(peers);
});

ws.on("CATCH_UP", ({ ops }) => {
  // Apply all missed operations in order
  ops.forEach(({ op, version, userId }) => {
    if (version > currentVersion) {
      applyOperation(op);
      currentVersion = version;
    }
  });
});

ws.on("OPERATION", ({ op, version, userId }) => {
  if (version !== currentVersion + 1) {
    // Out of order: request catch-up
    ws.send({ type: "JOIN", docId, lastSeenVersion: currentVersion });
    return;
  }
  // Transform against any pending local operations not yet acknowledged
  const transformedOp = pendingOps.reduce(
    (transformed, localOp) => transformOperation(transformed, localOp),
    op,
  );
  applyOperation(transformedOp);
  currentVersion = version;
});
```

</details>

---

## 🔗 Related Topics

- [`networking/01-http-protocols.md`](./01-http-protocols.md) — HTTP that SSE runs over
- [`networking/02-fetch-and-xhr.md`](./02-fetch-and-xhr.md) — Fetch for SSE implementation
- [`javascript-core/13-service-workers.md`](../javascript-core/13-service-workers.md) — SW as a WebSocket proxy
- [`caching/04-data-caching.md`](../caching/04-data-caching.md) — Real-time cache updates from WebSocket
- [`system-design/06-event-driven-frontend.md`](../system-design/06-event-driven-frontend.md) — Event-driven patterns with WebSocket

---

<div align="center">

**Next:** [`networking/04-cors-and-security.md`](./04-cors-and-security.md) →

</div>
