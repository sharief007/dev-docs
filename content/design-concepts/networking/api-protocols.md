---
title: REST vs gRPC vs GraphQL
weight: 11
type: docs
toc: true
sidebar:
  open: true
prev:
next:
params:
  editURL:
---

Three dominant API paradigms used at FAANG — each optimized for a different problem. Knowing when to reach for each is a system design interview staple.

## At a Glance

| | REST | gRPC | GraphQL |
|---|---|---|---|
| **Transport** | HTTP/1.1 or HTTP/2 | HTTP/2 (required) | HTTP/1.1 or HTTP/2 |
| **Wire format** | JSON (text) | Protocol Buffers (binary) | JSON (text) |
| **Schema** | OpenAPI (optional) | `.proto` (required) | SDL (required) |
| **Payload size** | Large (verbose JSON) | Small (binary, ~3–10× smaller) | Variable (client-specified) |
| **Latency** | Moderate | Low | Moderate to high (resolver fan-out) |
| **Streaming** | SSE / chunked | ✅ Native (4 patterns) | ✅ Subscriptions (WebSocket) |
| **Caching** | ✅ Easy (HTTP cache headers) | ❌ Hard (POST-only, no URL) | ❌ Hard (POST-only, dynamic queries) |
| **Browser support** | ✅ Native | ⚠️ Requires gRPC-Web proxy | ✅ Native |
| **Type safety** | Optional (OpenAPI codegen) | ✅ Enforced at compile time | ✅ Schema-validated at runtime |
| **Versioning** | URL path (`/v2/`) or header | Field addition (backward compat) | Schema evolution with `@deprecated` |

## REST

REST (Representational State Transfer) maps operations to HTTP verbs on resource URLs. The server is stateless — no session state between requests.

**HTTP verb semantics:**

| Verb | Semantics | Idempotent | Safe |
|------|-----------|-----------|------|
| GET | Read | ✅ | ✅ |
| POST | Create / trigger | ❌ | ❌ |
| PUT | Replace (full update) | ✅ | ❌ |
| PATCH | Partial update | ❌ (unless designed so) | ❌ |
| DELETE | Delete | ✅ | ❌ |

**Idempotency** matters at scale — retrying a safe/idempotent operation is always safe. Retrying a POST (non-idempotent) may create duplicate records. Use idempotency keys (`Idempotency-Key: <uuid>`) for POST endpoints that should be retry-safe (payment APIs, order creation).

**HTTP caching** is REST's biggest advantage — `GET` responses are cacheable by default. Reverse proxies (Nginx, Varnish) and CDNs cache based on URL + headers automatically. `ETag` and `Last-Modified` enable conditional requests to avoid sending unchanged data.

**Over-fetching and under-fetching:** A single REST endpoint returns a fixed shape. A mobile client asking for a user's name gets the full user object (over-fetch). Assembling a feed requires multiple round trips to `/users`, `/posts`, `/comments` (under-fetch). This is the problem GraphQL was designed to solve.

**Versioning trap:** URL versioning (`/v1/`, `/v2/`) duplicates controller logic and is hard to deprecate. Header versioning (`Accept: application/vnd.api.v2+json`) is cleaner but less visible. Additive changes (new optional fields) avoid versioning entirely — prefer this when possible.

## gRPC

gRPC uses HTTP/2 as the transport and Protocol Buffers as the serialization format. The API contract lives in a `.proto` file; client and server code is generated from it.

**Proto definition:**

```protobuf
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);           // Unary
  rpc StreamOrders (StreamRequest) returns (stream Order); // Server streaming
  rpc UploadItems (stream Item) returns (UploadResult);    // Client streaming
  rpc Chat (stream Message) returns (stream Message);      // Bidirectional
}

message Order {
  string order_id = 1;
  int64  created_at = 2;
  repeated LineItem items = 3;
}
```

**Four communication patterns:**

| Pattern | Client sends | Server sends | Use case |
|---------|-------------|-------------|---------|
| Unary | 1 request | 1 response | Standard RPC call |
| Server streaming | 1 request | stream of responses | Live feed, large result set |
| Client streaming | stream of requests | 1 response | File upload, telemetry ingest |
| Bidirectional | stream | stream | Chat, collaborative editing |

**Why binary matters at scale:** A JSON payload of 1 KB becomes ~100–300 bytes in Protobuf. At 100k RPS, that's 70–90 MB/s of bandwidth saved. More importantly, binary parsing is significantly faster than JSON — fewer CPU cycles per request.

**Deadlines and cancellation:** gRPC has first-class deadline propagation. A client sets a deadline; the server checks `ctx.Done()` and cancels in-progress work. Deadlines cascade through service calls — if the root request deadline expires, all downstream gRPC calls are cancelled. This prevents cascading slow-drain failures.

```go
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel()
resp, err := client.GetOrder(ctx, &pb.GetOrderRequest{OrderId: "123"})
```

**Browser limitation:** Browsers cannot speak HTTP/2 trailers, which gRPC requires. **gRPC-Web** solves this with a JavaScript client that communicates with an Envoy proxy (or Nginx module) that translates to native gRPC. This adds an extra hop and loses bidirectional streaming.

## GraphQL

GraphQL exposes a single endpoint. The client sends a query specifying exactly which fields it needs. The server returns only those fields.

```graphql
# Client query — asks only for what it needs
query {
  user(id: "u_123") {
    name
    avatar
    recentOrders(limit: 3) {
      id
      total
      status
    }
  }
}
```

The server resolves each field via a **resolver function**. Fields can be resolved from different data sources (databases, microservices, caches).

**N+1 problem:** A naive resolver for `recentOrders` fires one DB query per user. Fetching a feed of 100 users triggers 1 (users) + 100 (orders) = 101 queries.

```
users query        → 1 SQL query   → returns 100 users
orders resolver    → 100 SQL queries (one per user)   ← N+1
```

**Fix: DataLoader batching.** DataLoader collects resolver calls within a single event loop tick, batches them into one query, then distributes results.

```
orders resolver    → batched: SELECT * FROM orders WHERE user_id IN (u1, u2, ..., u100)
                   → 1 SQL query regardless of result size
```

**Caching is hard:** REST uses URL-based HTTP caching naturally. GraphQL queries are POST requests — no URL to cache on. Solutions:

| Approach | How it works |
|----------|-------------|
| **Persisted queries** | Client registers query hash; server stores query by hash. GET `/graphql?queryId=abc123` — now cacheable by CDN |
| **Response cache** (Apollo) | Cache full query responses by hash in Redis |
| **CDN caching** | Only works with persisted queries over GET; dynamic queries cannot be CDN-cached |
| **Fragment caching** | Cache individual resolver results, not full responses |

**Schema federation (Apollo Federation):** Large orgs split the schema across teams. Each service owns its subgraph. The gateway stitches subgraphs into a unified schema at query time. Netflix, Shopify, and Twitter use this pattern to let product teams own their own GraphQL types independently.

{{< callout type="warning" >}}
GraphQL introspection — the ability to query the schema itself — is useful in development but should be disabled in production for public APIs. It exposes your full data model and can be used to map attack surface.
{{< /callout >}}

## Decision Guide

| Scenario | Recommendation | Reason |
|----------|---------------|--------|
| Public-facing API (third-party developers) | REST | Familiar, widely tooled, easy to cache, works in every HTTP client |
| Internal microservice communication | gRPC | Binary efficiency, compile-time contracts, deadline propagation, streaming |
| Mobile BFF (Backend for Frontend) | GraphQL | Mobile clients fetch exactly the fields they render — avoids over-fetch on metered connections |
| Multi-client (web, iOS, Android, TV) | GraphQL | One schema serves all clients; each client queries its own shape |
| High-throughput data pipeline | gRPC | Binary payload, streaming, low CPU overhead |
| Real-time data (live scores, collaborative) | gRPC bidirectional or WebSocket over REST | Native streaming patterns |
| Startup / small team | REST | Simplest to build, debug, and evolve; no codegen step |
| Service mesh (Istio, Linkerd) | gRPC | Sidecar proxies speak HTTP/2 natively; richer observability (per-RPC metrics) |

{{< callout type="info" >}}
These are not mutually exclusive. A common pattern at FAANG: **gRPC** between internal microservices, **REST** for the public API gateway, and **GraphQL** as the BFF layer that aggregates internal gRPC calls into client-optimized responses.
{{< /callout >}}
