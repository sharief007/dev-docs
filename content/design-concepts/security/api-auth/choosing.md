---
title: Choosing the Right Scheme
weight: 5
type: docs
---

## Choosing the Right Scheme: Decision Framework

```mermaid
flowchart TB
    Start([API Authentication<br/>Decision]) --> Q1{Who is the caller?}

    Q1 -->|"End user<br/>via browser/mobile"| OAuth[OAuth 2.0 + PKCE<br/>→ JWT Bearer Token]

    Q1 -->|"External partner<br/>server-to-server"| Q2{Need request<br/>integrity?}
    Q2 -->|"Yes — financial,<br/>webhook"| HMAC[API Key +<br/>HMAC Signing]
    Q2 -->|"No — simple<br/>read API"| AK[API Key<br/>in header]

    Q1 -->|"Internal<br/>microservice"| Q3{Infrastructure<br/>maturity?}
    Q3 -->|"Service mesh<br/>or Vault"| MTLS[mTLS<br/>certificate-based]
    Q3 -->|"Simple setup"| JWTINT[JWT with<br/>short expiry]

    Q1 -->|"Third-party<br/>OAuth integration"| CC[OAuth 2.0<br/>Client Credentials]

    style OAuth fill:#bfb
    style HMAC fill:#bfb
    style AK fill:#ffb
    style MTLS fill:#bfb
    style JWTINT fill:#bfb
    style CC fill:#bfb
```

### Head-to-Head Comparison

| Property | API Key | HMAC Signing | mTLS | JWT Bearer |
|----------|---------|-------------|------|-----------|
| **Identity proof** | Application ID | Application ID + request integrity | Service identity (certificate) | User + application identity |
| **Request integrity** | No | Yes (signed body + path + timestamp) | Yes (TLS encryption) | No (token only, not request body) |
| **Replay protection** | No | Yes (timestamp window) | Yes (TLS session) | Partial (expiry, but within window) |
| **Setup complexity** | Minimal | Moderate (signing logic on client) | High (CA, cert distribution, rotation) | Moderate (auth server, JWKS) |
| **Revocation** | Delete key from DB | Delete key from DB | Revoke certificate (or wait for expiry) | Short expiry + refresh token revocation |
| **Credential exposure risk** | Key in every request header | Key never sent — only signature | Certificate on disk, auto-rotated | Token in every request header |
| **Validation cost** | DB lookup per request | DB lookup + HMAC computation | TLS handshake (then session reuse) | Local crypto only (~50µs) |

### Real-World Patterns

| Scenario | Recommended scheme | Example |
|----------|-------------------|---------|
| **Public API for developers** | API key (identity) + OAuth 2.0 (user data access) | Google Maps API key + OAuth for user data |
| **Webhook receiver** | HMAC signature verification | Stripe webhook `Stripe-Signature` header |
| **Internal microservices** | mTLS (service mesh) or JWT service tokens | Istio sidecar proxies handle mTLS transparently |
| **Mobile app accessing user data** | OAuth 2.0 Authorization Code + PKCE → JWT Bearer | Instagram API, Spotify API |
| **Partner server integration** | API key + HMAC signing | AWS S3 API (Signature V4) |
| **CLI tool or CI/CD pipeline** | OAuth 2.0 Device Flow or short-lived API token | `gh auth login`, `aws configure` |
| **Service-to-service (no user)** | OAuth 2.0 Client Credentials → JWT | Backend billing service calling user API |

### Layering Schemes Together

In practice, production APIs often **combine** multiple schemes:

```mermaid
flowchart LR
    subgraph "External Request"
        R1[API Key<br/>identifies the app] --> R2[OAuth JWT<br/>identifies the user]
        R2 --> R3[Rate limit<br/>per API key]
    end

    subgraph "Webhook Callback"
        W1[HMAC Signature<br/>proves authenticity] --> W2[Timestamp<br/>prevents replay]
    end

    subgraph "Internal Service Call"
        I1[mTLS<br/>proves service identity] --> I2[JWT<br/>carries user context]
    end
```

**Example: Stripe's model**

- **Public API calls:** secret API key (`sk_live_...`) sent via HTTP Basic auth (`Authorization: Basic <base64(sk_live_...:)>`) identifies the merchant account + authenticates
- **Webhook deliveries to your server:** HMAC-SHA256 signature in `Stripe-Signature` header, verified with your webhook secret
- **User-initiated actions:** OAuth 2.0 Connect for platform integrations, JWT for session management

{{< callout type="info" >}}
**Interview tip:** When API authentication comes up, say: "I'd choose the scheme based on the caller type. For user-facing requests from a mobile app, OAuth 2.0 with PKCE gives me delegated auth and scoped JWT Bearer tokens — validated locally at the API gateway with zero DB calls. For external partner integrations that modify data, I'd use API keys for identity plus HMAC request signing — the partner signs the method, path, timestamp, and body hash with their secret key, which prevents tampering and replay attacks. For internal service-to-service calls, mTLS with short-lived certificates from an internal CA gives me mutual identity verification at the transport layer — the service mesh handles it transparently. API keys are stored hashed (like passwords), JWTs are validated by signature check against JWKS, and HMAC uses constant-time comparison to prevent timing attacks." This shows you match the scheme to the trust model and understand the security details.
{{< /callout >}}

## Test Your Understanding

{{< details title="You run a public developer API. Each call must (a) identify which app is calling (for rate limits and billing) and (b) access a specific end user's private data. Why is a single scheme the wrong answer?" closed="true" >}}
**The two requirements are about two different principals, so you layer two schemes.** An **API key** identifies the *application* — perfect for rate-limiting and billing per client — but it says nothing about the *user* and can't safely grant access to a user's private data. For that you need **OAuth 2.0**: the user delegates scoped access and your app receives a JWT Bearer token representing *that user*.

**Combined:** API key (app identity + quota) + OAuth/JWT (user identity + scoped data access). Google Maps-style keys plus OAuth for user data is the canonical example.
{{< /details >}}

{{< details title="Stripe sends webhooks to your server. There's no bearer token or API key from Stripe in the request — so how do you authenticate that the webhook is genuinely from Stripe and not a spoof?" closed="true" >}}
**Verify the HMAC signature Stripe attaches, using your shared webhook secret.** Stripe signs the raw payload (plus a timestamp) with a secret only you and Stripe know and sends it in the `Stripe-Signature` header. You recompute `HMAC-SHA256(payload, webhook_secret)` and compare (constant-time). A spoofed request can't produce a valid signature without the secret, and the timestamp bounds replay.

**Why not mTLS or a bearer token here?** The caller is *them* calling *you* — you can't hand Stripe a token, and you don't control their client certs. A shared-secret signature is the natural fit for verifying inbound webhooks.
{{< /details >}}
