---
title: HTTP Status Codes
weight: 7
type: docs
---

Your monitoring dashboard alerts on a spike of 502s — what does that actually tell you? It says your reverse proxy can't reach the backend, which is very different from a 503 (your server is overloaded) or a 504 (your backend is just slow). Status codes are how the entire web — browsers, CDNs, load balancers, retry libraries, monitoring tools — make automatic decisions about what to do next. Picking the wrong one breaks retry logic, caching behavior, and on-call dashboards.

Status codes are three-digit integers in the response status line. The first digit defines the class.

```
HTTP/1.1 404 Not Found
         ↑↑↑
         class = 4 (client error)
```

| Class | Range | Meaning |
|-------|-------|---------|
| **1xx** | 100–199 | Informational — request received, continue |
| **2xx** | 200–299 | Success — request understood and processed |
| **3xx** | 300–399 | Redirection — further action required |
| **4xx** | 400–499 | Client Error — bad request from the client |
| **5xx** | 500–599 | Server Error — server failed a valid request |

## 1xx — Informational

| Code | Name | Description |
|------|------|-------------|
| **100** | Continue | Server received request headers; client should send the body. Used with `Expect: 100-continue` to avoid sending large bodies on auth failure. |
| **101** | Switching Protocols | Server is switching protocols per client request. Used to upgrade from HTTP/1.1 → WebSocket or HTTP/2. |
| **103** | Early Hints | Server sends preliminary response headers before the final response. Allows client to preload resources while server prepares the full response. |

## 2xx — Success

| Code | Name | Description |
|------|------|-------------|
| **200** | OK | Standard success. Body present. Returned for GET, POST, PUT, PATCH. |
| **201** | Created | Resource successfully created. `Location` header should point to the new resource. Returned for POST/PUT. |
| **202** | Accepted | Request accepted for processing, but processing not complete. Used for async jobs. No body required. |
| **204** | No Content | Success, no body. Common for DELETE and PUT responses where the client doesn't need the updated resource. |
| **206** | Partial Content | Response contains a partial resource body. Used with `Range` requests for resumable downloads and video scrubbing. |

## 3xx — Redirection

The client must take additional action to complete the request.

| Code | Name | Permanent? | Method Preserved? | Description |
|------|------|:---:|:---:|-------------|
| **301** | Moved Permanently | ✅ | ❌ (may change to GET) | Resource permanently at new URI. Browsers and crawlers update bookmarks/indexes. |
| **302** | Found | ❌ | ❌ (may change to GET) | Temporary redirect. Original URI should still be used for future requests. |
| **303** | See Other | ❌ | ❌ (always GET) | Redirect after a POST to a GET resource. Prevents re-submission on refresh. POST → redirect → GET pattern. |
| **304** | Not Modified | — | — | Resource not changed since `If-Modified-Since` or `If-None-Match`. Client uses cached version. No body returned. |
| **307** | Temporary Redirect | ❌ | ✅ | Temporary redirect; method and body preserved. Use instead of 302 if method must not change. |
| **308** | Permanent Redirect | ✅ | ✅ | Permanent redirect; method and body preserved. Use instead of 301 if method must not change. |

### 301 vs 302 vs 307 vs 308

| | Permanent | Method Preserved |
|---|:---:|:---:|
| 301 | ✅ | ❌ |
| 302 | ❌ | ❌ |
| 307 | ❌ | ✅ |
| 308 | ✅ | ✅ |

{{< callout type="info" >}}
Use **308** when permanently moving a POST/PUT endpoint. Use **307** for temporary redirects that must preserve the HTTP method. When in doubt between 301 and 302, prefer 308 and 307 respectively — they are strictly correct.
{{< /callout >}}

{{< callout type="info" >}}
**304 returns no body.** The browser uses its cached copy. `Content-Type`, `Content-Length`, and `Content-Encoding` headers are also omitted. Triggered by conditional request headers (`If-None-Match`, `If-Modified-Since`).
{{< /callout >}}

## 4xx — Client Error

The request is invalid or the client lacks permission.

| Code | Name | Description |
|------|------|-------------|
| **400** | Bad Request | Malformed syntax, invalid parameters, validation failure. Most common catch-all for bad input. |
| **401** | Unauthorized | Authentication required or failed. Response must include a `WWW-Authenticate` header describing the scheme. |
| **403** | Forbidden | Authenticated but not authorized. The server understands the request but refuses it. |
| **404** | Not Found | Resource does not exist at this URI. Also used intentionally to hide existence of unauthorized resources. |
| **405** | Method Not Allowed | HTTP method not supported on this resource. Response must include `Allow` header listing valid methods. |
| **408** | Request Timeout | Server timed out waiting for the client to send the full request. |
| **409** | Conflict | State conflict, e.g., duplicate entry, concurrent write conflict, or version mismatch. |
| **410** | Gone | Resource permanently deleted and will not return. Unlike 404, explicitly signals the resource existed. |
| **411** | Length Required | Server requires `Content-Length` header; client didn't send one. |
| **413** | Content Too Large | Request body exceeds the server's limit. Common for file upload endpoints. |
| **415** | Unsupported Media Type | Server can't process the request body's `Content-Type`. |
| **422** | Unprocessable Entity | Request is syntactically valid but semantically incorrect (e.g., failed business validation). Common in REST APIs. |
| **429** | Too Many Requests | Rate limit exceeded. Response should include `Retry-After` header. |

### 401 vs 403

| | 401 | 403 |
|---|-----|-----|
| Identity known? | No (or failed auth) | Yes |
| Can re-auth help? | ✅ | ❌ |
| Meaning | "Who are you?" | "I know you, but no." |

## 5xx — Server Error

The server failed to fulfill a valid request.

| Code | Name | Description |
|------|------|-------------|
| **500** | Internal Server Error | Generic server-side failure. Unhandled exception. Default error when nothing else fits. |
| **501** | Not Implemented | Server doesn't support the requested method. Different from 405 (method exists but not on this resource). |
| **502** | Bad Gateway | Upstream server returned an invalid response. Seen when a reverse proxy (Nginx, API Gateway) can't reach the backend. |
| **503** | Service Unavailable | Server temporarily unable to handle requests — overloaded or in maintenance. Include `Retry-After` if possible. |
| **504** | Gateway Timeout | Upstream server didn't respond in time. Seen at load balancers and API gateways when backend is slow/unresponsive. |
| **507** | Insufficient Storage | Server can't store the representation (WebDAV). Occasionally used for quota-exceeded scenarios. |

### 500 vs 502 vs 503 vs 504

| Code | Who failed | When you see it |
|------|-----------|-----------------|
| 500 | Application code | Unhandled exception, crash |
| 502 | Upstream server | Invalid/corrupt response from backend |
| 503 | This server | Overloaded, in deploy/restart |
| 504 | Upstream server | Backend too slow (timeout) |

## Quick Reference

| Code | Typical API Use |
|------|----------------|
| 200 | Successful GET, POST, PUT |
| 201 | Resource created (POST) |
| 204 | Delete succeeded / update with no body |
| 206 | Partial file download, video range request |
| 301 | Permanent domain or URL change |
| 304 | Conditional GET cache hit |
| 400 | Validation error, malformed JSON |
| 401 | Missing or invalid token |
| 403 | Valid token, wrong permissions |
| 404 | Resource not found |
| 409 | Duplicate record, optimistic lock conflict |
| 422 | Semantic validation failure |
| 429 | Rate limit hit |
| 500 | Unhandled server exception |
| 502 | Reverse proxy can't reach backend |
| 503 | Server overloaded or deploying |
| 504 | Backend timed out |

{{< callout type="info" >}}
**Interview tip:** When asked to design API error semantics, be precise: "I'd map 401 to 'who are you' (authentication missing or invalid) and 403 to 'I know you, but no' (authenticated but unauthorized) — and always include `WWW-Authenticate` on 401 so clients know which scheme to retry with. For validation, 400 means malformed syntax and 422 means semantically invalid (failed business rule) — separating them lets clients distinguish parse errors from rule violations. For redirects on POST/PUT endpoints I'd use 307 or 308 instead of 302/301 because they preserve method and body. For server errors, 502 means upstream returned garbage, 503 means I'm overloaded (include `Retry-After`), 504 means upstream timed out — distinct codes drive distinct on-call runbooks. And for rate limiting I'd return 429 with `Retry-After` so clients can back off cleanly instead of hammering."
{{< /callout >}}
