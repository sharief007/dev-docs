---
title: 'Rate Limiting'
weight: 1
type: docs
toc: true
prev: 
next: 
params:
  editURL:
---

### What ?
A mechanism to control the number of requests a client can make to a server/API within a time window. Excess requests are delayed, queued, or blocked to prevent server overload.

### Why ?
#### Benefits:
- **DDoS Protection**: Prevents resource exhaustion from attacks.
- **Cost Control**: Manages usage-based billing (e.g., cloud services).
- **Server Health**: Prevents server overload and ensures stability.

#### Drawbacks:
- **User Experience**: Can frustrate users if limits are too restrictive.

### Where ?

- Client Side Implementation: 
    - **Unreliable**: Requests can be forged.
    - **No Control**: Cannot enforce on external clients.
- Server Side Implementation: Implemented in the API, typically returns status `429` (Too Many Requests).
- Middleware Implementation:  API gateway supports rate limiting, SSL termination, authentication, IP whitelisting, servicing static content, etc.

### How ?
#### Rate Limiting Algorithms
- [Token Bucket Algorithm](token-bucket)
- [Leaking bucket]()
- [Fixed window counter]()
- [Sliding window log]()
- [Sliding window counter]()