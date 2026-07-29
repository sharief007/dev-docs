---
title: Tokens in Practice
weight: 5
type: docs
---

## Token Size Considerations

JWTs grow with every claim you add. Since the token is sent in the `Authorization` header of **every request**, size matters:

```
Minimal JWT (sub + exp only):     ~200 bytes
Typical JWT (standard claims):     ~400 bytes
Heavy JWT (roles, permissions):    ~800 bytes
Excessive JWT (full user profile): ~2 KB+

HTTP header size limits:
  - Most servers: 8 KB default (nginx, Apache)
  - AWS ALB: 16 KB total headers
  - Cloudflare: 16 KB total headers

At 2 KB per token × 100K requests/second = 200 MB/s of bandwidth just for auth headers
```

{{< callout type="warning" >}}
**Keep JWTs lean.** Include only the claims needed for authorization decisions (user ID, role, scopes). Do not embed full user profiles, permission lists, or org hierarchies. If a service needs detailed user data, fetch it from a user service using the `sub` claim — don't bloat every HTTP request.
{{< /callout >}}

## JWT vs Opaque Tokens vs Sessions

| Property | JWT (self-contained) | Opaque token (reference) | Server-side session |
|----------|---------------------|--------------------------|-------------------|
| **Validation** | Local (cryptographic) | Remote (introspection endpoint or DB lookup) | Remote (session store lookup) |
| **Latency** | ~50µs (CPU only) | +1–20ms (network call) | +0.5–2ms (Redis call) |
| **Revocation** | Hard (need blocklist) | Easy (delete from DB) | Easy (delete from store) |
| **Scalability** | Excellent (stateless) | Good (centralized token store) | Moderate (shared session store) |
| **Token size** | 400–800 bytes | 32–64 bytes (just a random string) | 32 bytes (session ID cookie) |
| **Data in token** | Claims visible to client | Nothing — server looks up data | Nothing — server looks up data |
| **Best for** | Microservices, cross-domain APIs, mobile apps | When instant revocation is required | Traditional server-rendered web apps |

{{< callout type="info" >}}
**Interview tip:** When discussing authentication in a system design interview, say: "I'd use JWTs for stateless authentication. The auth server issues a short-lived RS256-signed JWT (15-minute expiry) containing the user ID, role, and audience claim. API servers validate the token locally by checking the signature against the public key cached from the JWKS endpoint — no DB or Redis round-trip per request. RS256 over HS256 because only the auth server needs the private key; compromising an API server doesn't let attackers forge tokens. I'd always validate `aud` to prevent cross-service token misuse, hardcode the algorithm allowlist to prevent the `alg: none` and key-confusion attacks, and keep the payload minimal — just `sub`, `role`, `scope`, `exp`, and `aud`. For revocation, short token TTL handles most cases; for instant revocation (compromised account), I'd add a Redis-backed JTI blocklist checked by the API gateway." This covers the mechanism, key management, all three pitfalls, and the revocation trade-off — exactly what interviewers probe.
{{< /callout >}}

## Test Your Understanding

{{< details title="A team embeds the user's full profile and a 200-entry permission list in the JWT, producing a ~3 KB token. Auth passes in tests. What fails in production?" closed="true" >}}
**The token rides on every request, so the bloat compounds — and can breach header limits.** At ~3 KB per token times 100K req/s that's ~300 MB/s of bandwidth spent just carrying auth headers. Worse, many servers/proxies cap total header size (nginx ~8 KB, ALB/Cloudflare ~16 KB); combined with cookies and other headers, a fat JWT can trigger `431 Request Header Fields Too Large` or silent truncation.

**Fix:** keep the token lean — `sub`, `role`/`scopes`, `exp`, `aud`. If a service needs the full profile or fine-grained permissions, look them up by `sub` from a user/authorization service instead of shipping them in every request.
{{< /details >}}

{{< details title="JWTs validate faster (no lookup) than opaque tokens. So when would you deliberately choose an opaque token instead?" closed="true" >}}
**When instant, reliable revocation matters more than validation speed.** An opaque token is just a random reference string; the server looks up its state on each call, so revoking it is a single delete that takes effect immediately — ideal for high-value sessions (banking, admin consoles) or public APIs where you must be able to kill a token *now*.

**The trade-off you accept:** every request pays a lookup (introspection endpoint or DB) — the very cost JWTs avoid. Many systems run both: JWTs for user APIs, reference tokens where revocation is non-negotiable.
{{< /details >}}
