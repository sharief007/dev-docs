---
title: Grant Types
weight: 2
type: docs
---

## Grant Types: Matching the Flow to the Client

OAuth 2.0 defines multiple "grant types" — each is a different protocol flow optimized for a specific type of client. Using the wrong grant type creates security vulnerabilities.

### Authorization Code Grant (User-Facing Web/Mobile Apps)

This is the **most common and most secure** flow for applications where a user is present. The key security property: the access token is never exposed to the user's browser.

#### The Problem It Solves

A web application needs to act on behalf of a user (e.g., read their Google Calendar). The app must prove to Google that the user consented, but the app's backend — not the browser — should hold the sensitive token.

#### The Flow

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant App as Your App (Backend)
    participant AS as Auth Server (Google)
    participant API as Resource Server (Google Calendar)

    U->>App: Click "Sign in with Google"
    App->>U: Redirect to Google's auth page

    U->>AS: GET /authorize?<br/>response_type=code&<br/>client_id=abc&<br/>redirect_uri=https://app.com/callback&<br/>scope=calendar.read&<br/>state=xyz123

    Note over U,AS: User sees Google's consent screen:<br/>"App wants to read your calendar"

    U->>AS: User clicks "Allow"
    AS->>U: Redirect to https://app.com/callback?code=AUTH_CODE&state=xyz123

    U->>App: GET /callback?code=AUTH_CODE&state=xyz123

    Note over App: Verify state matches to prevent CSRF

    App->>AS: POST /token<br/>grant_type=authorization_code&<br/>code=AUTH_CODE&<br/>client_id=abc&<br/>client_secret=SECRET&<br/>redirect_uri=https://app.com/callback

    AS-->>App: {access_token, refresh_token, expires_in: 900}

    Note over App: Token stored server-side — never reaches the browser

    App->>API: GET /calendar/events<br/>Authorization: Bearer access_token
    API-->>App: Calendar events
    App-->>U: Rendered calendar page
```

**Why the extra "code" step?** The authorization code is a short-lived, one-time-use intermediary. It's exchanged for the real token in a **back-channel** request (server-to-server) that includes the `client_secret`. This means:
- The access token is never in the browser's URL bar or history
- The `client_secret` proves the app's identity (a stolen auth code is useless without it)
- The exchange happens over a server-to-server HTTPS connection, not through the user's browser

### PKCE: Protecting Public Clients

#### The Problem It Solves

Mobile apps and single-page applications (SPAs) **cannot securely store a `client_secret`**. The app's code is on the user's device — any secret embedded in it can be extracted. Without a secret, a stolen authorization code can be exchanged by an attacker.

```
Attack without PKCE:

  1. User clicks "Login" in a mobile app
  2. OS opens browser → Google auth page → user approves
  3. Google redirects back to the app: myapp://callback?code=AUTH_CODE
  4. A malicious app registered for the same URL scheme intercepts the redirect
  5. Malicious app exchanges AUTH_CODE for an access token
  6. Malicious app now has access to the user's data
```

#### How PKCE Prevents This

PKCE (Proof Key for Code Exchange, pronounced "pixy") ties the authorization code to the specific client that requested it, without needing a stored secret.

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant AS as Auth Server

    Note over App: Generate random code_verifier (43-128 chars)<br/>code_challenge = SHA256(code_verifier)

    App->>AS: GET /authorize?<br/>code_challenge=HASH&<br/>code_challenge_method=S256&<br/>...other params

    Note over AS: Store code_challenge with this auth session

    AS-->>App: callback?code=AUTH_CODE

    App->>AS: POST /token<br/>code=AUTH_CODE&<br/>code_verifier=ORIGINAL_RANDOM_STRING

    Note over AS: Compute SHA256(code_verifier)<br/>Compare with stored code_challenge<br/>Match → issue token

    AS-->>App: {access_token, refresh_token}
```

**Why the attacker can't use a stolen code:** The malicious app intercepts the `AUTH_CODE` but doesn't have the `code_verifier` (it was generated in memory by the legitimate app and never transmitted). Without the verifier, the token exchange fails.

### Client Credentials Grant (Service-to-Service)

#### The Problem It Solves

A backend microservice needs to call another internal API. There is no user involved — the service itself is the client and needs to authenticate as itself.

```mermaid
sequenceDiagram
    participant Billing as Billing Service
    participant AS as Auth Server
    participant API as User API

    Billing->>AS: POST /token<br/>grant_type=client_credentials&<br/>client_id=billing-service&<br/>client_secret=SERVICE_SECRET&<br/>scope=users.read

    AS-->>Billing: {access_token, expires_in: 3600}

    Billing->>API: GET /users/123/subscription<br/>Authorization: Bearer access_token
    API-->>Billing: Subscription data
```

No browser, no redirect, no user consent. The service proves its identity with its `client_id` + `client_secret` (or mTLS certificate) and receives a token scoped to the permissions assigned to that service.

### Device Authorization Grant (TVs, CLIs, IoT)

#### The Problem It Solves

A smart TV, game console, or CLI tool has no browser and limited input capability. The user can't type a URL or interact with a web-based consent screen on the device itself.

```mermaid
sequenceDiagram
    participant TV as Smart TV
    participant AS as Auth Server
    participant Phone as User's Phone

    TV->>AS: POST /device/code<br/>client_id=tv-app&scope=streaming
    AS-->>TV: {device_code: "xyz", user_code: "ABCD-1234",<br/>verification_uri: "https://auth.example.com/device"}

    Note over TV: Display on screen:<br/>"Go to https://auth.example.com/device<br/>and enter code: ABCD-1234"

    Phone->>AS: User visits URL, enters "ABCD-1234"
    Phone->>AS: User approves on their phone

    loop Poll every 5 seconds
        TV->>AS: POST /token<br/>grant_type=device_code&<br/>device_code=xyz
        AS-->>TV: {error: "authorization_pending"}
    end

    Note over Phone,AS: User approves

    TV->>AS: POST /token (poll)
    AS-->>TV: {access_token, refresh_token}

    Note over TV: Now authenticated — can stream content
```

The device polls the auth server until the user completes authorization on a separate device with a full browser and keyboard.

## Test Your Understanding

{{< details title="In the Authorization Code flow, why add the extra 'code' step at all? Why not have the authorization server return the access token directly in the browser redirect?" closed="true" >}}
**To keep the token out of the browser.** The redirect URL is visible in the address bar, browser history, and server/referrer logs — returning a token there (the old **Implicit** flow) exposes it. Instead the server returns a short-lived, single-use **code**, which the app's *backend* exchanges for the token over a server-to-server back channel, including the `client_secret` (or PKCE verifier). The token itself never travels through the user agent.

**That's also why Implicit is deprecated:** Authorization Code + PKCE gives browser-only apps the same no-secret capability without ever putting a token in a URL.
{{< /details >}}

{{< details title="You're adding login to a smart TV app with no keyboard and no browser. Why is Authorization Code (even with PKCE) a poor fit, and what do you use instead?" closed="true" >}}
**The TV can't render a consent page or let the user type credentials — Authorization Code assumes a capable browser on the same device.** The **Device Authorization Grant** solves this: the TV shows a short code and a URL, the user approves on their *phone*, and the TV **polls** the token endpoint until authorization completes.

**The takeaway:** pick the grant by *client capability and trust*, not habit — Authorization Code + PKCE for browsers/mobile, Client Credentials for service-to-service, Device flow for input-constrained devices.
{{< /details >}}
