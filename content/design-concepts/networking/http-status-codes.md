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
**Interview tip:** "401 = 'who are you?' (auth missing), 403 = 'I know you, but no' (authorized but denied). For validation: 400 = malformed syntax, 422 = semantically invalid. Redirects on POST/PUT: use 307/308 (preserve method) instead of 301/302. Server errors: 502 = upstream garbage, 503 = I'm overloaded (include `Retry-After`), 504 = upstream timeout. Distinct codes drive distinct on-call runbooks."
{{< /callout >}}

## Test Your Understanding

{{< details title="Your API returns 200 for everything — success and errors — with an error field in the JSON body. A teammate says this is fine because 'clients parse the body anyway.' What breaks?" closed="true" >}}
**Everything that automates based on status codes breaks:**

1. **CDN caching** — CDNs cache 200 responses. Your error response gets cached and served to other users.
2. **Retry logic** — HTTP client libraries and middleware retry on 5xx but not on 2xx. Your errors never trigger retries.
3. **Monitoring/alerting** — dashboards count 5xx rates. Your errors show as 0% error rate despite real failures.
4. **Load balancer health checks** — passive health checks count 5xx responses to detect failed backends. Your "200 with error" bypasses detection.
5. **Browser behavior** — `304 Not Modified` saves bandwidth via conditional requests. If everything is 200, conditional caching doesn't work.

HTTP status codes are a **machine-readable API contract**, not just a human convention.
{{< /details >}}

{{< details title="A client receives a 301 redirect from POST /api/v1/orders to /api/v2/orders. The client follows the redirect, but the order isn't created. What happened?" closed="true" >}}
**301 (and 302) do not preserve the HTTP method.** Per spec, browsers and many HTTP clients convert the redirected request from POST to GET. The client follows the redirect with `GET /api/v2/orders` instead of `POST /api/v2/orders` — no request body, no order created.

**Fix:** Use **308 Permanent Redirect** (or 307 for temporary). Both preserve the original method and body. The client follows the redirect with `POST /api/v2/orders` and the order is created.

This is a real-world bug that has broken payment flows and form submissions. The 307/308 codes were specifically created to fix this ambiguity in 301/302.
{{< /details >}}

{{< details title="Your monitoring alerts on 502 errors. The on-call engineer checks the backend and it's running fine — healthy, responding to local requests. Where's the problem?" closed="true" >}}
**502 Bad Gateway means the reverse proxy/load balancer received an invalid response from the backend**, not that the backend is down. Possible causes:

1. **Backend returned a malformed HTTP response** — missing headers, truncated body, or non-HTTP response on an HTTP port
2. **Connection reset by backend** — backend accepted the connection but closed it before sending a response (crash during processing, timeout, or connection limit hit)
3. **Protocol mismatch** — the proxy expects HTTP/1.1 but the backend speaks HTTP/2 (or vice versa). Nginx's `proxy_pass` doesn't support HTTP/2 to backends — use `grpc_pass` for gRPC.
4. **Backend timeout + proxy timeout misconfiguration** — the backend timed out internally and sent a partial response that the proxy couldn't parse

The backend appears "healthy" to local requests because the issue is in the **proxy ↔ backend** connection path, not the backend itself. Check proxy error logs, not just backend health checks.
{{< /details >}}

{{< details title="A rate-limited client receives 429 Too Many Requests but keeps retrying immediately, making the overload worse. What header should you include, and what retry strategy should the client implement?" closed="true" >}}
Include **`Retry-After: N`** (seconds to wait) or `Retry-After: <HTTP-date>`. This tells the client exactly when it's safe to retry.

The client should implement **exponential backoff with jitter:**
1. First retry: wait `Retry-After` seconds (or 1s if absent)
2. Subsequent retries: double the wait time (2s, 4s, 8s, ...) up to a max (e.g., 60s)
3. Add **random jitter** (±20%) to prevent synchronized retry storms when many clients are rate-limited at the same time

**Without jitter:** If 1000 clients all get 429 at the same time and all retry after exactly 1 second, you get another spike of 1000 requests at t+1s. Jitter spreads retries across [0.8s, 1.2s], smoothing the retry wave.
{{< /details >}}

{{< details title="Your API returns 404 for unauthorized resources instead of 403. A security auditor flags this. Is the auditor right?" closed="true" >}}
**Both approaches are valid — it depends on the threat model.**

**Returning 403:** Confirms the resource **exists** but the user lacks permission. An attacker can enumerate resource IDs by distinguishing 403 (exists, no access) from 404 (doesn't exist).

**Returning 404:** Hides the existence of resources the user can't access. The attacker can't distinguish "doesn't exist" from "exists but forbidden." This is called **resource enumeration protection** and is the approach GitHub, AWS, and most cloud APIs use for sensitive resources.

**The auditor is wrong if** the API intentionally uses 404 for security (documented behavior). **The auditor is right if** the API inconsistently returns 403 for some unauthorized resources and 404 for others — the inconsistency leaks information about which resources exist.
{{< /details >}}
