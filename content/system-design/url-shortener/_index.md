---
title: 'URL Shortener'
weight: 1
type: docs
---

You are asked to design a URL shortening service like TinyURL or Bitly. A user submits a long URL such as `https://example.com/very/long/path?with=query&params=here` and receives a short link like `https://sho.rt/aB3xZ9`. When anyone visits the short link, they are redirected to the original URL.

The service looks trivial — "just store a key and a value" — but the interesting part is scale: a popular shortener serves **billions of redirects per day** with a read:write ratio in the thousands, must keep redirect latency in the low tens of milliseconds globally, and must generate short, collision-free keys across a fleet of servers without coordinating on every request.

## Functional Requirements

1. **Shorten:** Given a long URL, return a unique short URL. The short key should be as short as practical (7 characters).
2. **Redirect:** Given a short URL, redirect the client to the original long URL.
3. **Custom alias:** Optionally let the user pick a custom key (e.g. `sho.rt/my-brand`).
4. **Expiration:** Optionally attach a TTL; expired links stop resolving.
5. **Analytics (basic):** Count clicks per short URL (approximate is acceptable).

## Out of Scope

- User accounts, authentication, and billing (assume an upstream API-key gateway handles identity).
- Rich analytics dashboards, geographic/referrer breakdowns beyond a click counter.
- Malware/phishing URL scanning (would be a separate pipeline).
- Link editing after creation (short keys are immutable).

## Non-Functional Requirements

- **Scale:** 100M new URLs/month; 10B redirects/day. Read:write ≈ **3000:1** (extremely read-heavy).
- **Latency:** Redirect p99 **< 50 ms** end-to-end; shorten p99 < 200 ms.
- **Availability:** **99.9%** for redirects (a dead redirect breaks every embedded link). Creation can tolerate slightly lower availability.
- **Consistency:** Redirects may be **eventually consistent** — a newly created link resolving a few hundred ms later is fine. Key uniqueness must be **strongly** guaranteed (never hand out the same key twice).
- **Durability:** Links are effectively permanent (default 5-year retention); losing the mapping is unacceptable.
- **Scalability of keyspace:** 7 base62 characters = 62⁷ ≈ **3.5 trillion** keys, comfortably above 100M/month × many years.
