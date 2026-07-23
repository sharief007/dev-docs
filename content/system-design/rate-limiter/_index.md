---
title: 'Rate Limiter Service'
weight: 1
type: docs
---

Every API gateway has a common adversary: unconstrained clients. A single misbehaving caller — a buggy app caught in a retry loop, an aggressive scraper, or a paying customer running an unthrottled batch job — can saturate backend capacity and degrade service for everyone else. A **rate limiter** is the enforcement layer that caps how many requests a client may make in a given time window and rejects excess traffic with HTTP 429 before it reaches origin services.

At small scale a rate limiter is trivial: one in-memory counter per user, checked on every request. At system-design interview scale the hard problems are distributional: a fleet of API gateway nodes must enforce limits *across all nodes* with sub-millisecond overhead on every single request. Getting this right requires choosing the right counting algorithm, executing counter operations atomically against a shared store, and carefully managing the three-way tension between accuracy, latency, and Redis load.

## Functional Requirements

1. **Enforce rate limits per-user** (for authenticated requests), **per-IP** (for unauthenticated requests), and **per-endpoint** — a single request may be simultaneously subject to a global user quota *and* a tighter endpoint-specific quota.
2. **Reject excess requests** with HTTP 429 Too Many Requests and standard response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After`.
3. **Allow short bursts** above the sustained limit for well-behaved clients, governed by a per-tier burst multiplier.
4. **Support configurable limit tiers** — free plan, paid plan, internal service exemption — that change without a code deploy.
5. **Enforce limits at the API gateway layer**, before any request reaches a backend origin service.

## Out of Scope

- Authentication and identity resolution — handled upstream; the rate limiter receives an already-resolved `user_id`.
- DDoS / volumetric attack mitigation — a network / CDN-layer concern operating orders of magnitude higher in the stack.
- Business-level quotas (e.g. "max 100 payment transactions per day") — belong in the domain service, not the gateway.
- Rate limiting between internal microservices — covered by service-mesh policy, not the API gateway.

## Non-Functional Requirements

- **Scale:** 1M req/s sustained across the gateway fleet; burst peaks up to 3M req/s. Up to 10M distinct users generating traffic within any given minute.
- **Latency:** rate-limit enforcement must add **< 1 ms p99** overhead per request. Blocking the hot path for a slow I/O round-trip is unacceptable.
- **Availability:** **99.99%** (< 1 hour downtime/year). The rate limiter must **fail-open** rather than become a single point of failure — a degraded enforcer should not take the entire API offline.
- **Consistency:** **eventually consistent** — a brief window of over-allowance (a handful of extra requests slipping through across nodes within a sync interval) is acceptable. Exact precision is traded for performance.
- **Durability:** counter state is transient. Losing a few seconds of in-progress counts on a Redis restart is acceptable — rate-limiting state is not durable business data.
- **Cardinality:** limits apply at (identifier × endpoint) granularity — up to 10M users × ~100 endpoints = 1B logical counter keys, though only a fraction are active at any moment.
