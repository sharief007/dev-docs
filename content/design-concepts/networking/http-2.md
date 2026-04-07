---
title: HTTP/2
weight: 4
type: docs
sidebar:
  open: true
---

HTTP/2 (RFC 7540, 2015) keeps the same semantics as HTTP/1.1 — methods, status codes, headers — but replaces the text-based wire format with a **binary framing layer**. The primary goals: eliminate HOL blocking at the application layer, reduce header overhead, and allow multiple requests to share a single TCP connection.

## Binary Framing Layer

HTTP/1.1 sends requests and responses as ASCII text. HTTP/2 splits every message into **frames** — small binary units that can be interleaved and reassembled.

```
HTTP/1.1 (text)                HTTP/2 (binary frames)
──────────────────             ──────────────────────
GET /index.html HTTP/1.1  →    HEADERS frame
Host: example.com              DATA frame(s)
Accept: text/html
[blank line]
```

### Frame Structure

Every frame starts with a fixed 9-byte header:

```
┌──────────────────────────────────────┐
│  Length: 24 bits  │  Type: 8 bits    │
├──────────────────────────────────────┤
│  Flags: 8 bits    │  Stream ID: 31 b │
├──────────────────────────────────────┤
│  Payload (variable, up to 16KB–16MB) │
└──────────────────────────────────────┘
```

| Frame Type | Purpose |
|------------|---------|
| `HEADERS` | Initiates a stream; carries request/response headers (HPACK-compressed) |
| `DATA` | Carries body bytes for a stream |
| `SETTINGS` | Negotiates connection parameters (max frame size, header table size, etc.) |
| `WINDOW_UPDATE` | Flow control — grants permission to send more data |
| `PUSH_PROMISE` | Server announces a resource it will push before being asked |
| `RST_STREAM` | Immediately terminate a single stream without closing the connection |
| `GOAWAY` | Gracefully close the connection; signals last processed stream ID |
| `PING` | Measure round-trip time; keep connection alive |

## Streams and Multiplexing

A **stream** is a logical bidirectional channel within a single TCP connection. Each stream carries one request/response pair and has a unique integer ID.

- Client-initiated streams: **odd IDs** (1, 3, 5, …)
- Server-initiated streams (push): **even IDs** (2, 4, 6, …)
- Stream 0: reserved for connection-level control frames (SETTINGS, PING, GOAWAY)

Multiple streams are **interleaved** on the same connection — frames from different streams can be mixed freely:

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    Note over C,S: Single TCP connection — streams interleaved

    C->>S: HEADERS [Stream 1] GET /index.html
    C->>S: HEADERS [Stream 3] GET /style.css
    C->>S: HEADERS [Stream 5] GET /script.js

    S->>C: HEADERS [Stream 3] 200 OK
    S->>C: DATA   [Stream 3] (CSS — ready first)
    S->>C: DATA   [Stream 1] (HTML chunk 1)
    S->>C: HEADERS [Stream 5] 200 OK
    S->>C: DATA   [Stream 5] (JS)
    S->>C: DATA   [Stream 1] (HTML chunk 2 — last)
```

{{< callout type="info" >}}
This eliminates **application-level HOL blocking**. In HTTP/1.1, a slow response blocked everything behind it. In HTTP/2, Stream 1 being slow has no effect on Streams 3 or 5.
{{< /callout >}}

**Key properties:**
- Default max concurrent streams: 100 (negotiated via `SETTINGS_MAX_CONCURRENT_STREAMS`)
- Browsers typically use **one TCP connection per origin** (vs 6 in HTTP/1.1)
- Streams can be opened and closed independently without affecting the connection

## HPACK Header Compression

HTTP/1.1 sends headers as plain text on every request. A typical request carries 500–2000 bytes of headers (cookies, user-agent, accept-*) even when most of them haven't changed.

HPACK (RFC 7541) compresses headers using two techniques:

### Static Table

A predefined table of 61 common header name/value pairs. Sending the index number instead of the literal string:

| Index | Header | Value |
|-------|--------|-------|
| 2 | `:method` | `GET` |
| 3 | `:method` | `POST` |
| 4 | `:path` | `/` |
| 7 | `:scheme` | `https` |
| 8 | `:status` | `200` |
| 13 | `content-length` | |
| 23 | `content-type` | `application/json` |

Sending `GET /` over HTTPS takes **2 bytes** (indices 2 and 7) instead of the full text.

### Dynamic Table

Both client and server maintain a table that grows as new headers are seen. A header sent once can be referenced by index in all subsequent requests. The table is **per-connection** — entries are lost when the connection closes.

```
Request 1: Authorization: Bearer eyJhbGc...  → sent as literal, added to dynamic table at index 62
Request 2: Authorization: Bearer eyJhbGc...  → sent as index 62 (single byte)
```

### Huffman Encoding

Literal values that can't be indexed are Huffman-encoded — common characters use fewer bits. Average compression: **~85–95% reduction** in header bytes on subsequent requests.

{{< callout type="warning" >}}
Compression oracle attacks (CRIME targeted DEFLATE inside TLS/SPDY; BREACH targeted gzip-compressed HTTP response bodies) exploit the fact that secret bytes compress more efficiently when they appear alongside known plaintext. HPACK uses Huffman encoding — not DEFLATE — so it is not directly vulnerable to those specific attacks. However, the same class of attack applies if attacker-controlled values can be injected into the same compression context as secrets. Never mix user-controlled input with sensitive values in the same compressed header block.
{{< /callout >}}

## Server Push

The server can proactively send resources the client will need — before the client asks for them.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: GET /index.html [Stream 1]
    S->>C: PUSH_PROMISE [Stream 2] — "I'll send /style.css"
    S->>C: PUSH_PROMISE [Stream 4] — "I'll send /script.js"
    S->>C: HEADERS + DATA [Stream 1] — HTML response
    S->>C: HEADERS + DATA [Stream 2] — CSS (pushed)
    S->>C: HEADERS + DATA [Stream 4] — JS (pushed)

    Note over C: Client already has /style.css cached
    C->>S: RST_STREAM [Stream 2] — cancel push
```

**When it helps:** Server knows from the HTML response that the client will need CSS and JS. By pushing them immediately, it eliminates a round trip.

**Limitations in practice:**
- Server can't know if the client already has the resource cached
- Pushed resources can't be shared across tabs (unlike cached responses)
- Client can cancel with `RST_STREAM` but bandwidth for the pushed data is already spent
- **Deprecated in Chrome (2022)** and removed from Firefox; replaced by `103 Early Hints`

{{< callout type="info" >}}
Use **`Link: </style.css>; rel=preload`** in response headers or `103 Early Hints` instead of server push. The browser can then decide whether to fetch based on its cache.
{{< /callout >}}

## Flow Control

Each stream and the connection itself has a **receive window** — the maximum amount of unacknowledged `DATA` frames the sender can transmit.

- Default window: 65,535 bytes (per stream and connection)
- Receiver sends `WINDOW_UPDATE` to grant more capacity
- Prevents a fast sender from overwhelming a slow receiver
- Operates at both stream level and connection level independently

## HTTP/2 Limitations

| Problem | Why it persists |
|---------|----------------|
| TCP-level HOL blocking | All streams share one TCP connection; a single dropped packet stalls every stream |
| Slow start | New TCP connections start with a small congestion window; takes time to ramp up |
| Connection migration breaks | TCP connections are tied to IP:port; a network change (WiFi → cellular) kills the connection |
| Handshake latency | TCP 3-way handshake + TLS 1.3 = minimum 2 RTT before first byte |

{{< callout type="warning" >}}
**TCP-level HOL blocking is worse in HTTP/2 than HTTP/1.1.** HTTP/1.1 opens 6 connections — a packet loss on one affects only ~1/6 of requests. HTTP/2 uses one connection — a packet loss stalls 100% of in-flight streams simultaneously.
{{< /callout >}}

## HTTP/1.1 vs HTTP/2

| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| Wire format | Text (ASCII) | Binary frames |
| Connections per origin | 6 (browser workaround) | 1 |
| Request multiplexing | ❌ | ✅ |
| HOL blocking (app level) | ✅ (pipelining) | ❌ |
| HOL blocking (TCP level) | ✅ | ✅ |
| Header compression | ❌ (plaintext, repeated) | ✅ (HPACK) |
| Server push | ❌ | ✅ (deprecated in practice) |
| TLS required | No | No (but enforced by all browsers) |
