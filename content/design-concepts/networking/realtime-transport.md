---
title: WebSockets vs Long Polling vs SSE
weight: 10
type: docs
---

You're building a stock-trading app. Prices need to update on screen the instant they change on the exchange — but HTTP only lets the client ask, not the server tell. Polling every 100ms wastes bandwidth and hammers your servers; polling every 5s shows stale prices. The fix depends on what direction the data flows: SSE for one-way price updates, WebSockets for the bidirectional order entry channel, and long polling as the firewall fallback. Pick wrong and your scaling story falls apart at 100K concurrent users.

HTTP is request-response — the client always initiates. Real-time features (notifications, live feeds, chat) need the server to push data to the client. Three patterns exist, each with different tradeoffs.

## The Evolution of Server Push

```
Short Polling          Long Polling            SSE                    WebSocket
                                           (HTTP stream)          (protocol upgrade)

C ──GET /poll──► S     C ──GET /poll──► S    C ──GET /events──► S   C ──Upgrade──► S
C ◄── 204 ────── S     S holds open          S ◄────────────────    ◄══════════════►
(repeat every Ns)      S ◄── 200 (data) ─    S sends chunks         (full duplex)
                       C ──GET /poll──► S     as they arrive         continuously
                       (reconnect)
```

Short polling wastes requests — the server almost always has nothing new. Long polling and SSE are HTTP-based; WebSocket is a separate protocol.

## Long Polling

The client sends a request. The server holds the connection open until it has data to send (or a timeout fires), then responds. The client immediately sends another request.

```
Client                          Server
  │──── GET /notifications ────►│
  │                             │ (holds open — 29s elapsed)
  │◄─── 200 {"msg": "new order"}│
  │──── GET /notifications ────►│ (immediately reconnects)
  │                             │ (timeout — 30s)
  │◄─── 204 No Content ─────────│
  │──── GET /notifications ────►│
```

**Key properties:**
- Standard HTTP — works through every proxy, firewall, load balancer
- One in-flight request per client at all times
- Reconnect overhead: each cycle pays TCP/TLS setup (unless keep-alive reuses the connection)
- Server must correlate the reconnecting client back to its state

**Where long polling is still used:** Twilio, Stripe webhooks fallback, environments where WebSockets are blocked by firewalls.

## Server-Sent Events (SSE)

The client makes a single HTTP GET. The server responds with `Content-Type: text/event-stream` and keeps the response body open, writing events as they occur. The client never closes the connection voluntarily.

```
GET /events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: {"type":"price_update","symbol":"AAPL","price":189.42}\n\n

event: alert\n
data: {"msg":"order filled"}\n\n

: heartbeat\n\n
```

**Wire format:**

| Field | Meaning |
|-------|---------|
| `data:` | Event payload (one line per field, blank line terminates event) |
| `event:` | Named event type (client uses `addEventListener('alert', ...)`) |
| `id:` | Last-event-ID; sent as `Last-Event-ID` header on reconnect |
| `retry:` | Tells client how many ms to wait before reconnecting |
| `: comment` | Heartbeat / keep-alive (ignored by client, prevents proxy timeout) |

**Auto-reconnect:** The browser's `EventSource` API reconnects automatically with `Last-Event-ID`, allowing the server to resume from where the stream left off.

**HTTP/2 advantage:** Each SSE subscription is one HTTP/2 stream — many subscriptions share a single TCP connection. Under HTTP/1.1, browsers cap connections at 6 per origin, limiting concurrent SSE streams.

## WebSockets

WebSocket starts as HTTP, then upgrades to a persistent full-duplex TCP connection. Either side can send frames at any time.

**Upgrade handshake:**

```
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the 101, the connection is no longer HTTP. The server and client exchange **frames**, not requests/responses.

**Frame types:**

| Opcode | Purpose |
|--------|---------|
| `0x1` Text | UTF-8 payload (JSON messages) |
| `0x2` Binary | Raw bytes (Protobuf, MessagePack, audio) |
| `0x8` Close | Graceful shutdown with status code |
| `0x9` Ping | Keepalive probe |
| `0xA` Pong | Keepalive reply |

**Key properties:**
- Full-duplex — server and client both push without waiting
- Binary support — efficient for audio, video, game state
- No built-in auto-reconnect — application must implement
- Each connection is a stateful, persistent TCP socket (important for scaling)

## Side-by-Side Comparison

| | Long Polling | SSE | WebSocket |
|---|---|---|---|
| **Direction** | Server → Client | Server → Client | Bidirectional |
| **Protocol** | HTTP | HTTP | ws:// / wss:// |
| **Persistent connection** | No (reconnects each cycle) | Yes | Yes |
| **Browser API** | `fetch` / `XMLHttpRequest` | `EventSource` | `WebSocket` |
| **Auto-reconnect** | Manual | ✅ Built-in (`EventSource`) | Manual |
| **HTTP/2 multiplexing** | ✅ | ✅ | ❌ (separate TCP) |
| **Binary support** | ❌ | ❌ (text only) | ✅ |
| **Proxy / firewall friendly** | ✅ (plain HTTP) | ✅ (plain HTTP) | Sometimes blocked |
| **Load balancer support** | ✅ | ✅ | Requires sticky sessions |
| **Overhead per message** | High (HTTP headers each cycle) | Low (chunked stream) | Very low (2–10 byte frame header) |

## Scaling WebSocket Connections

WebSocket connections are **stateful** — a persistent TCP socket exists between the client and a specific server process. This breaks horizontal scaling assumptions.

**Problem: message fan-out across instances**

```mermaid
sequenceDiagram
    participant CA as Client A
    participant S1 as Server 1
    participant R as Redis Pub/Sub
    participant S2 as Server 2
    participant CC as Client C

    CA->>S1: WebSocket connect (room:chat)
    CC->>S2: WebSocket connect (room:chat)
    Note over S1: SUBSCRIBE room:chat
    Note over S2: SUBSCRIBE room:chat

    CC->>S2: Send "Hello everyone"
    S2->>R: PUBLISH room:chat "Hello everyone"
    R-->>S1: Delivers to subscriber
    R-->>S2: Delivers to subscriber (self)
    S1->>CA: Push "Hello everyone"
    S2->>CC: Push "Hello everyone"
```

Each server subscribes to Redis (or Kafka, NATS) channels. When a message arrives on any server, it publishes to the bus; all other servers deliver it to their connected clients.

**Sticky sessions:** Without pub/sub, the load balancer must route a client to the same server every time (sticky by IP or session cookie). This creates uneven load and complicates deploys.

{{< callout type="warning" >}}
A single WebSocket server process typically handles 10k–100k concurrent connections before hitting file descriptor limits or memory pressure. Plan connection counts early — a chat app with 1M online users needs ~10–100 WebSocket server processes.
{{< /callout >}}

**SSE scaling is simpler:** SSE is stateless from the load balancer's perspective — any server can serve an SSE stream as long as it can subscribe to the same event source (database, message bus). No sticky sessions required.

## When to Use Each

| Use case | Best fit | Reason |
|----------|----------|--------|
| Live notifications (new email, order update) | SSE | Server→client only; HTTP/2 multiplexing; auto-reconnect |
| Live dashboard / stock ticker | SSE | Continuous server push; no client→server messages needed |
| Chat / collaborative editing | WebSocket | Bidirectional — client and server both send frequently |
| Multiplayer game state | WebSocket | Binary frames, low overhead, low latency |
| Live sports scores | SSE | Broadcast to many clients; server push only |
| Presence indicators ("X is typing") | WebSocket | Client must push events to server |
| Environment blocks WebSockets (corporate proxy) | SSE or Long Polling | HTTP-based protocols bypass WS restrictions |
| IoT telemetry ingest | WebSocket | Binary, persistent, bidirectional for command & control |

{{< callout type="info" >}}
**Interview tip:** When asked how to design real-time features, pick the protocol from the data direction: "If it's server-to-client only — notifications, live dashboards, stock tickers — I'd use SSE. It's plain HTTP so it works through every proxy and load balancer, the browser's `EventSource` API auto-reconnects with `Last-Event-ID` for free, and over HTTP/2 each subscription is a stream so one TCP connection multiplexes thousands of subscribers. For bidirectional flows like chat or collaborative editing I'd use WebSockets and accept the cost: connections are stateful so the LB needs sticky sessions, a single server typically tops out at 10K–100K concurrent connections, and message fan-out across instances requires Redis or NATS pub/sub. I'd reserve long polling for environments where corporate proxies block WebSockets — Stripe and Twilio still use it as a fallback. The scaling math I'd flag: a chat app with 1M online users needs ~10–100 WebSocket processes, and externalize all session state because losing a node loses every connected client."
{{< /callout >}}
