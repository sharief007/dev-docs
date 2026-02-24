---
title: Load Balancing Algorithms
weight: 2
type: docs
toc: true
sidebar:
  open: true
prev: layer-4-vs-layer-7
next:
params:
  editURL:
---

The load balancer selects a backend for each incoming request. The algorithm determines which backend is chosen and when.

## Algorithms

| Algorithm | How it selects | Use when | Gotcha |
|-----------|---------------|----------|--------|
| **Round Robin** | Sequential rotation across all backends | Uniform request cost, homogeneous backends | Doesn't account for request duration — a slow in-flight request still receives the next rotation |
| **Weighted Round Robin** | Sequential, but backends receive traffic proportional to assigned weight | Heterogeneous backends (different CPU/RAM) | Weights are static; requires manual tuning when capacity changes |
| **Least Connections** | Backend with the fewest active connections | Variable request duration (mix of fast and slow requests) | Tracks connection count, not actual load — a backend handling 10 heavy queries looks same as 10 lightweight ones |
| **Least Response Time** | Backend with fewest connections AND lowest avg response time | Latency-sensitive; heterogeneous workloads | Response time is a trailing average — slow to react to sudden degradation |
| **IP Hash** | `hash(client_ip) % n` determines backend | Simple session affinity, no cookie support | Adding or removing a backend reshuffles most hashes — use consistent hashing instead |
| **Consistent Hashing** | Hash ring with virtual nodes; each backend owns a range | Cache backends, session affinity, stateful routing | Requires virtual nodes (100–200 per backend) to prevent uneven distribution |
| **Random** | Random backend selection | High-throughput, stateless; simple to implement | Performs surprisingly well — approaches least-connections as request count grows |
| **Power of Two Choices (P2C)** | Pick 2 random backends, route to the one with fewer connections | Stateless services at scale | Requires connection count tracking; not available in all proxies (Envoy, Linkerd) |

### Round Robin vs Least Connections

Round Robin distributes by count, not by cost:

```
Backend A: handles 10 fast requests (10ms each) → 100ms total work
Backend B: handles 1 slow request (1000ms)       → 1000ms total work
Round Robin gives backend B the next request.    ← wrong choice
Least Connections routes to backend A.           ← correct choice
```

Use Least Connections whenever requests have variable processing time.

### IP Hash vs Consistent Hashing

IP hash (`hash(client_ip) % n`) breaks when topology changes:

```
3 backends: client 1.2.3.4 → hash=7 → 7%3=1 → backend B
Add 1 backend (n=4):        → hash=7 → 7%4=3 → backend D  ← session lost
```

Consistent hashing places backends on a hash ring. Adding/removing a backend remaps only `1/N` of keys:

```
Hash ring (0 to 2³²):
  Backend A owns: [0, 1B)
  Backend B owns: [1B, 2B)
  Backend C owns: [2B, 3B)
  Backend D owns: [3B, 4B)

Add Backend E between B and C:
  Backend E owns: [1.5B, 2B)  ← only B's former range is affected
  Everything else: unchanged
```

Virtual nodes: each backend is placed at multiple points on the ring (100–200 virtual nodes per backend) to prevent uneven key distribution when backends have unequal ring coverage.

### Power of Two Choices (P2C)

Pick 2 backends at random, route to the one with fewer connections.

- Avoids the "thundering herd" problem of pure Least Connections: with LC, every proxy independently picks the same "best" backend → that backend gets overloaded simultaneously
- P2C: each proxy picks a different random pair → load distributes naturally
- Achieves near-optimal load distribution with O(1) selection cost
- Used by Envoy and Linkerd as the default algorithm

## Session Affinity (Sticky Sessions)

Some applications store session state in memory on a specific backend. Sticky sessions ensure a client always routes to the same backend.

| Method | How | Tradeoff |
|--------|-----|----------|
| **IP hash** | Hash client IP to select backend | Breaks with NAT — many clients share one IP; CG-NAT routes all mobile users from one ISP to the same backend |
| **Cookie injection** | Proxy sets a cookie (e.g., `SERVERID=app1`); subsequent requests carry it | Reliable; but requires cookie support and HTTP (not available in L4 mode) |
| **Consistent hashing on session ID** | Hash a request attribute (session ID header, user ID) | Most flexible; proxy reads the attribute and routes deterministically; works even when backends change |

{{< callout type="warning" >}}
Sticky sessions couple state to a specific instance. If that instance dies, all its sessions are lost. The correct architecture is **externalizing session state** (Redis, database) so any backend can serve any request. Sticky sessions are a workaround for stateful applications that cannot be refactored.
{{< /callout >}}

## Choosing an Algorithm

| Scenario | Algorithm |
|----------|-----------|
| Identical backends, uniform request cost | Round Robin |
| Identical backends, variable request cost | Least Connections or P2C |
| Different backend capacities | Weighted Round Robin |
| Cache tier — same key must hit same backend | Consistent Hashing |
| High-throughput, stateless, simplicity required | Random |
| Microservices sidecar proxy at scale | Power of Two Choices |
| Stateful app that can't be refactored | Cookie-based sticky sessions |
