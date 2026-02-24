---
title: Layer 4 vs Layer 7 Load Balancing
weight: 1
type: docs
toc: true
sidebar:
  open: true
prev: _index
next: algorithms
params:
  editURL:
---

The layer determines what information the load balancer can see and act on. More information enables smarter routing but costs more CPU and memory.

## What Each Layer Sees

| | Layer 4 (Transport) | Layer 7 (Application) |
|---|---|---|
| IP source / destination | ✅ | ✅ |
| TCP/UDP port | ✅ | ✅ |
| TCP connection state (SYN, FIN, RST) | ✅ | ✅ |
| TLS SNI hostname | ✅ (passively, from ClientHello) | ✅ |
| HTTP method, URL, path | ❌ | ✅ |
| HTTP headers (Host, Cookie, Authorization) | ❌ | ✅ |
| Request body | ❌ | ✅ (with buffering) |
| TLS payload (decrypted) | ❌ | ✅ (after termination) |

## Connection Model

{{< tabs items="Layer 4,Layer 7" >}}
  {{< tab >}}
  Layer 4 uses **NAT (network address translation)** or **DSR (direct server return)**. The client's TCP connection goes directly to the backend — the LB rewrites IP headers and forwards packets.

  ```
  Client ──[TCP conn]──► L4 LB ──[same TCP conn, rewritten IP]──► Backend
                           ↑
                  Does NOT terminate the connection
                  Does NOT read the payload
                  Rewrites destination IP to backend IP
  ```

  - One TCP connection end-to-end (client ↔ backend, via LB)
  - LB maintains a connection table (src IP:port → backend IP:port)
  - No request buffering — packets forwarded as received
  - Backend sees the LB's IP (unless PROXY protocol or DSR is used)
  {{< /tab >}}

  {{< tab >}}
  Layer 7 uses **terminate-and-reoriginate**. The LB terminates the client connection and opens a new one to the backend.

  ```
  Client ──[TCP conn A]──► L7 LB ──[TCP conn B]──► Backend
                             ↑
                  Terminates conn A (reads full request)
                  Opens new conn B to selected backend
                  Two separate TCP connections
  ```

  - Two TCP connections: client→LB and LB→backend
  - LB parses the full HTTP request before routing
  - Full request body buffered before backend connection opens
  - Connection pool to backends (pre-warmed, reused across client requests)
  {{< /tab >}}
{{< /tabs >}}

## Performance Characteristics

| | Layer 4 | Layer 7 |
|---|---|---|
| Latency added | ~microseconds (packet rewrite only) | ~milliseconds (parse HTTP, buffer body, new TCP conn) |
| Throughput | Near wire speed | Limited by CPU for TLS + HTTP parsing |
| Memory per connection | Small (connection table entry) | Larger (HTTP parser state, request buffer, pool entry) |
| CPU cost | Minimal | Moderate to high (TLS handshake, header parsing) |
| Max connections | Tens of millions | Hundreds of thousands to low millions |

## Routing Capabilities

| Routing basis | Layer 4 | Layer 7 |
|---------------|---------|---------|
| Round robin / weighted | ✅ | ✅ |
| Least connections | ✅ (TCP connection count) | ✅ (HTTP request count) |
| IP-based | ✅ | ✅ |
| URL path (`/api/*` → service A) | ❌ | ✅ |
| HTTP header (Host, Cookie) | ❌ | ✅ |
| TLS SNI hostname | ✅ (passthrough only) | ✅ (after termination) |
| Canary / A-B routing by % traffic | ❌ | ✅ |
| Blue/green deployment routing | ❌ | ✅ |

## TLS Handling

{{< tabs items="L4 TLS Passthrough,L7 TLS Termination" >}}
  {{< tab >}}
  The L4 LB forwards encrypted TLS bytes directly to backends. Backends terminate TLS themselves.

  ```
  Client ──[TLS]──► L4 LB ──[TLS, unchanged]──► Backend
                       ↑
              Sees SNI hostname in ClientHello
              (used for routing, not decryption)
              Forwards encrypted bytes
  ```

  **Pros:** End-to-end encryption; LB never sees plaintext; backends own their certificates.

  **Cons:** LB cannot inspect HTTP headers or route based on URL; no centralized cert management; each backend needs its own certificate.
  {{< /tab >}}

  {{< tab >}}
  The L7 LB terminates TLS and forwards plaintext HTTP to backends.

  ```
  Client ──[TLS]──► L7 LB ──[HTTP]──► Backend
                       ↑
              Decrypts, inspects, routes
              Centralized certificate management
              Backends receive plain HTTP
  ```

  **Pros:** Centralized cert management (one cert per domain); backends simpler (no TLS config); LB can inspect and route on HTTP content.

  **Cons:** Plaintext on the internal network (acceptable if network is trusted); LB is a decryption point — a security boundary.
  {{< /tab >}}
{{< /tabs >}}

## Use Cases

| Use L4 when | Use L7 when |
|-------------|-------------|
| Non-HTTP protocols (SMTP, MySQL, gRPC over raw TCP) | HTTP/HTTPS traffic with content-based routing |
| Maximum throughput with minimal latency overhead | URL path routing (`/api` → service A, `/static` → CDN) |
| TLS passthrough required (backends own certs) | TLS termination at the edge |
| LB itself needs HA (L4 in front of L7 proxies) | Microservices with host-based virtual hosting |
| UDP traffic (DNS, game servers, QUIC) | Canary deploys, A/B testing, blue/green |
| Very high connection counts (>1M concurrent) | Authentication, rate limiting, request transformation |

## Real Tools

| Tool | Layer | Notes |
|------|-------|-------|
| AWS NLB (Network Load Balancer) | L4 | TCP/UDP passthrough; preserves client IP natively; scales to millions of connections |
| AWS ALB (Application Load Balancer) | L7 | HTTP/HTTPS; URL and header routing; native WebSocket support |
| HAProxy `mode tcp` | L4 | TCP passthrough; PROXY protocol support |
| HAProxy `mode http` | L7 | Full HTTP inspection; cookie stickiness; ACL routing |
| Nginx (stream block) | L4 | TCP/UDP proxy; SNI-based routing for TLS passthrough |
| Nginx (http block) | L7 | Full reverse proxy; upstream connection pooling |
| Envoy | L7 | Service mesh sidecar; P2C algorithm; circuit breaking; gRPC-aware |
| Google Cloud NLB | L4 | Regional; direct passthrough; no connection termination |
| Google Cloud HTTPS LB | L7 | Global anycast; HTTP/2 and QUIC support |

## Layered Architecture

In production, L4 and L7 are often combined:

```
Internet
    │
    ▼
L4 Load Balancer (AWS NLB / HAProxy mode tcp)
    │  ← distributes TCP connections across L7 instances
    │  ← preserves client IP via PROXY protocol
    ▼
L7 Reverse Proxy cluster (Nginx / HAProxy mode http / Envoy)
    │  ← terminates TLS, parses HTTP, routes by URL/host
    │  ← connection pool to backends
    ▼
Backend Services
```

The L4 layer provides HA for the L7 proxies themselves. L4 has no single point of failure (stateless packet rewrite). The L7 layer handles application routing.

{{< callout type="info" >}}
This is why AWS separates NLB and ALB: use NLB in front of an ALB fleet for high availability of the ALB itself, or use NLB directly when you need TLS passthrough or non-HTTP protocols.
{{< /callout >}}
