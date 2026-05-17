---
title: CDN
weight: 9
type: docs
---

Your origin runs in `us-east-1`. A user in Sydney requests `logo.png` and pays 200ms RTT crossing the Pacific — even though that file hasn't changed in two years. Multiply that by every image, JS bundle, video segment, and you've got a slow, expensive site. Now put a Cloudflare PoP 5ms from that user in Sydney with the same logo cached at the edge: 200ms becomes 5ms, your origin egress bill drops 90%, and your origin survives Black Friday because most of the load never reaches it.

A CDN is a globally distributed network of servers (Points of Presence, PoPs) that cache and serve content close to the user. Every request that hits a CDN edge instead of your origin saves a round trip that might otherwise cross continents.

## How a Request Flows Through a CDN

```mermaid
sequenceDiagram
    participant U as User (NYC)
    participant E as CDN Edge (Newark PoP)
    participant O as Origin (us-east-1)

    U->>E: GET /logo.png
    Note over E: Cache HIT
    E->>U: 200 OK (from edge cache, ~5ms)

    U->>E: GET /api/feed
    Note over E: Cache MISS
    E->>O: GET /api/feed (forwarded to origin)
    O->>E: 200 OK
    Note over E: Response stored in edge cache
    E->>U: 200 OK (miss + origin RTT, ~50ms)
```

The edge returns a cache hit in single-digit milliseconds. A cache miss adds one extra hop to origin but still benefits from persistent connections and optimized routing.

### Cache Outcomes

| Outcome | Meaning | Response header |
|---------|---------|-----------------|
| **HIT** | Served from edge cache | `X-Cache: HIT` |
| **MISS** | Not in cache — fetched from origin, then cached | `X-Cache: MISS` |
| **BYPASS** | Intentionally not cached (e.g., authenticated request) | `X-Cache: BYPASS` |
| **EXPIRED** | In cache but TTL elapsed — revalidated with origin | `Age: 0` |
| **STALE** | Served stale while revalidation happens in background (`stale-while-revalidate`) | |

## Pull CDN vs Push CDN

| | Pull CDN | Push CDN |
|---|---|---|
| **How it works** | CDN fetches from origin on first cache miss | You upload content to CDN storage in advance |
| **Cache warm-up** | Cold on first request — origin takes the hit | Pre-warmed — first request is always a cache hit |
| **Best for** | Web assets, APIs, unknown access patterns | Large files (videos, firmware), predictable downloads |
| **Drawback** | First user after TTL expiry hits origin | Must push updates manually; stale content if you forget |
| **Examples** | Cloudflare, CloudFront (default) | CloudFront S3 origins, Akamai NetStorage |

Most web applications use pull CDNs. Push CDNs are used for video-on-demand and software distribution where you control the upload schedule.

## Cache Invalidation

Three mechanisms for removing stale content before TTL expires:

**TTL expiry** — set `Cache-Control: max-age=N`. Content is evicted automatically after N seconds. Simple, zero operational cost, but you wait out the TTL on bad deploys.

**Explicit purge** — call the CDN's purge API to evict by URL, tag, or prefix.

```
# Cloudflare — purge by URL
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer $TOKEN" \
  --data '{"files":["https://example.com/logo.png"]}'

# CloudFront — create invalidation
aws cloudfront create-invalidation \
  --distribution-id E1234 \
  --paths "/static/*"
```

**Cache-busting via versioned URLs** — embed a content hash in the filename. The URL changes on every deploy, so the old cached version and new version coexist without conflict.

```
# Old deploy
<script src="/static/main.abc123.js">

# New deploy — different URL, no purge needed
<script src="/static/main.def456.js">
```

Cache-busting is the most reliable strategy for immutable assets (JS bundles, CSS, images). Use explicit purge for content you can't version (HTML pages, API responses).

## Cache Key and Vary

The cache key is what the CDN uses to decide if a cached response can serve a new request. By default it's the full URL.

**Vary header** — tells the CDN to cache separate copies per request header value:

```
Vary: Accept-Encoding          # separate copies for gzip vs br vs identity
Vary: Accept-Language          # separate copy per locale
Vary: Authorization            # effectively disables caching (each token is unique)
```

{{< callout type="warning" >}}
`Vary: Authorization` or `Vary: Cookie` on dynamic endpoints means every unique token or session gets its own cache entry. The CDN fills up with uncacheable responses. Gate these responses with `Cache-Control: private` instead — the CDN will bypass and let the browser cache handle it.
{{< /callout >}}

## Origin Shield (Tiered Caching)

Without origin shield, every PoP has its own cache. A cache miss on 300 PoPs means 300 simultaneous requests to origin for the same resource (thundering herd after a deploy or TTL expiry).

```mermaid
graph LR
    subgraph without["Without Shield — 300 requests to origin"]
        P1[PoP 1] --> O1[Origin]
        P2[PoP 2] --> O1
        P3[PoP 300] --> O1
    end

    subgraph with["With Shield — 1 request to origin"]
        P4[PoP 1] --> S[Shield PoP]
        P5[PoP 2] --> S
        P6[PoP 300] --> S
        S --> O2[Origin]
    end
```

Origin shield adds a mid-tier cache layer. Edge PoPs that miss check the shield before hitting origin. One request to origin fills the shield; edge PoPs fill from the shield.

| Provider | Term |
|----------|------|
| CloudFront | Origin Shield |
| Cloudflare | Tiered Cache |
| Fastly | Shielding |
| Akamai | Tiered Distribution |

## Dynamic Content Acceleration

CDNs accelerate non-cacheable requests through network-level optimizations — the CDN still terminates the connection at the edge.

| Technique | What it does |
|-----------|-------------|
| **Persistent origin connections** | CDN PoP keeps a warm TCP connection to origin. Client avoids full TCP + TLS handshake to origin. |
| **Route optimization** | CDN uses private backbone or BGP optimization to route traffic faster than public internet |
| **TLS session resumption** | CDN terminates TLS at the edge (short RTT); reuses session ticket on CDN→origin leg |
| **Protocol upgrade** | Client ↔ CDN uses HTTP/2 or HTTP/3; CDN ↔ origin uses HTTP/2 over a pooled connection |

This is why API requests benefit from a CDN even when the response is not cached — the CDN PoP is the wall clock proximity optimization.

## CDN for Video Streaming

Video streaming is the highest-bandwidth CDN use case. The protocols (HLS, DASH) are designed around CDN caching.

**HLS (HTTP Live Streaming)** and **DASH (Dynamic Adaptive Streaming over HTTP)** both split video into small segments (2–6 seconds each) served as regular HTTP files. Each segment is independently cacheable.

```
manifest.m3u8         ← playlist file; short TTL (5–10s for live, longer for VOD)
  ├── seg-001.ts      ← video segment; immutable once written — long TTL (hours)
  ├── seg-002.ts
  └── ...
```

| Asset | Cache behavior |
|-------|---------------|
| Manifest (`*.m3u8`, `*.mpd`) | Short TTL — updates as new segments are added (live) or long TTL (VOD) |
| Segments (`*.ts`, `*.m4s`) | Immutable — infinite TTL, purge never needed |
| Thumbnail sprites | Long TTL |

**Multi-bitrate variant selection** — the manifest lists multiple quality levels (360p, 720p, 1080p, 4K). The player picks the variant based on available bandwidth. Each variant's segments are cached independently at the edge. The CDN does not participate in bitrate selection — it just serves whichever variant the player requests.

{{< callout type="info" >}}
For live streaming, the most recent segments are requested by every viewer within a few seconds of each other — exactly the thundering herd scenario. Origin shield is critical: without it, every PoP independently fetches the latest segment from origin for every viewer burst.
{{< /callout >}}

## Security at the Edge

| Capability | How CDNs use it |
|------------|----------------|
| **DDoS absorption** | CDN's aggregate bandwidth (Cloudflare: 280+ Tbps) absorbs volumetric attacks before they reach origin |
| **WAF** | Rules applied at edge — block SQLi, XSS, bad bots before request reaches your app |
| **Bot management** | Fingerprint and rate-limit scrapers, credential stuffing bots at the PoP |
| **TLS termination** | CDN handles TLS negotiation; origin can accept plain HTTP on a private network |
| **IP allowlisting at origin** | Lock origin to accept traffic only from CDN IP ranges — prevents origin bypass attacks |

{{< callout type="info" >}}
**Interview tip:** Separate static from dynamic: "For immutable assets I'd use content-hashed URLs with `Cache-Control: public, max-age=31536000, immutable` — versioned URLs eliminate invalidation problems. For cacheable API responses, shorter TTLs with explicit purge. The trap: `Vary: Authorization` creates a cache entry per token, filling the CDN with junk — use `Cache-Control: private` instead. For thundering herds after deploys, enable origin shield so 300 PoPs collapse to one origin request. Even non-cacheable APIs benefit from CDN: persistent origin connections, TLS termination at the edge, and DDoS absorption."
{{< /callout >}}

## Test Your Understanding

{{< details title="You deploy a new JS bundle but forgot to use content-hashed filenames. Users report seeing a mix of old and new behavior. How did this happen and what's the fix?" closed="true" >}}
The JS file URL (`/static/main.js`) is the same for old and new versions. CDN edge PoPs cached the old version at different times, so their TTLs expire at different times. Some PoPs serve the old bundle, others the new one — users get whichever their nearest PoP has.

**Immediate fix:** Purge `/static/main.js` across all PoPs via the CDN's purge API. But purge propagation isn't instant — some users may still get stale content from in-flight requests.

**Permanent fix:** Content-hashed filenames (`main.abc123.js`). Old and new versions have different URLs, so they coexist in cache without conflict. No purge needed — the HTML page references the new URL, and the old cached file just expires naturally.
{{< /details >}}

{{< details title="You set Cache-Control: public, max-age=3600 on an API endpoint that returns user-specific data. What's the security vulnerability?" closed="true" >}}
**Shared cache poisoning.** `public` tells the CDN to cache the response and serve it to **any** client. If User A's personalized response (name, email, account data) is cached, User B requesting the same URL gets User A's data.

**Fix:** Use `Cache-Control: private, no-store` for user-specific responses. `private` means only the browser may cache it (not CDN/proxies), and `no-store` prevents caching entirely. If you need CDN caching for authenticated endpoints, include a user-specific key in the cache key (e.g., via `Vary: Authorization` — but be aware this creates one cache entry per token, which may not be useful for CDN caching).
{{< /details >}}

{{< details title="Your live streaming platform has 100K concurrent viewers. A new video segment is published. Without origin shield, what happens to your origin server?" closed="true" >}}
Every CDN PoP that has viewers experiences a **cache miss** simultaneously for the new segment. If you have 300 PoPs, the origin receives ~300 near-simultaneous requests for the same segment (one per PoP). During a major live event, this thundering herd can overwhelm the origin.

**With origin shield:** All 300 PoPs first check the shield PoP. One request hits the origin, fills the shield, and all 300 PoPs fill from the shield. Origin sees 1 request instead of 300.

**Additional mitigation:** For live streaming, set very short TTLs on manifests (5–10s) but **long TTLs on segments** (segments are immutable once created). The manifest tells the player which segments exist; segments themselves never change.
{{< /details >}}

{{< details title="A CDN returns X-Cache: HIT but the response data is stale (3 hours old, TTL was 1 hour). How is this possible?" closed="true" >}}
**`stale-while-revalidate`.** If the origin set `Cache-Control: max-age=3600, stale-while-revalidate=7200`, the CDN can serve stale content for up to 2 additional hours **while** it revalidates with the origin in the background. The client gets a fast (but stale) response; the cache refreshes asynchronously.

**Other possibilities:**
- Origin was unreachable during revalidation, and the CDN is configured to serve stale on error (`stale-if-error`)
- Clock skew between CDN nodes caused incorrect TTL calculation
- The `Age` header (which tracks how long the response has been in cache) was stripped by a middle proxy, making the CDN think the response is fresher than it is
{{< /details >}}

{{< details title="Your API sets Vary: Accept-Encoding, Accept-Language. Users report slow responses despite high CDN cache hit rates on static assets. What's happening to API cache efficiency?" closed="true" >}}
`Vary` creates a **separate cache entry** for every unique combination of the specified headers. `Accept-Encoding` has ~3 variants (gzip, br, identity) — that's fine. But `Accept-Language` can have hundreds of variants (`en-US`, `en-GB`, `en`, `fr-FR`, `fr`, `de`, etc.).

3 encodings × N languages = **3N cache entries** per URL. If your API serves 50 locales, that's 150 cached copies of each response. Cache hit rate plummets because the cache is fragmented across too many variants.

**Fix:** Normalize `Accept-Language` to a small set of supported locales at the CDN edge (e.g., map `en-US`, `en-GB`, `en` → `en`). Or handle localization at the application level and cache a single locale-neutral response.
{{< /details >}}
