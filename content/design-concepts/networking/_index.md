---
title: Networking
weight: 11
type: docs
toc: false
sidebar:
  open: true
prev:
next:
params:
  editURL:
---

{{< cards >}}
    {{< card link="tcp-vs-udp" title="TCP vs UDP" subtitle="Handshake, flow control, congestion control, TIME_WAIT, SYN flood, and why QUIC chose UDP" >}}
    {{< card link="dns" title="DNS" subtitle="Resolution chain, record types, TTL, GeoDNS, load balancing, failover, and security" >}}
    {{< card link="http-1.1" title="HTTP/1.1" subtitle="Message structure, keep-alive, pipelining, and HOL blocking" >}}
    {{< card link="http-2" title="HTTP/2" subtitle="Binary framing, multiplexing, HPACK, and server push" >}}
    {{< card link="http-3" title="HTTP/3 and QUIC" subtitle="QUIC streams, 0-RTT, connection migration, and QPACK" >}}
    {{< card link="http-evolution" title="HTTP Evolution" subtitle="Protocol comparison, practical implications by client type, and CDN/proxy termination" >}}
    {{< card link="http-headers" title="HTTP Headers" subtitle="Request, response, security, and caching headers" >}}
    {{< card link="http-status-codes" title="HTTP Status Codes" subtitle="Complete reference for 1xx–5xx status codes" >}}
    {{< card link="reverse-proxy" title="Reverse Proxy vs Forward Proxy" subtitle="TLS termination, connection pooling, request buffering, circuit breaking, and header management" >}}
    {{< card link="load-balancing" title="Load Balancing" subtitle="L4 vs L7, algorithms (round-robin to P2C), health checks, sticky sessions, and global routing" >}}
    {{< card link="cdn" title="CDN" subtitle="Edge caching, pull vs push, cache invalidation, origin shield, and video streaming" >}}
    {{< card link="realtime-transport" title="WebSockets vs Long Polling vs SSE" subtitle="Protocol comparison, scaling stateful connections, and when to use each" >}}
    {{< card link="api-protocols" title="REST vs gRPC vs GraphQL" subtitle="Wire format, streaming patterns, caching tradeoffs, and decision guide" >}}
{{< /cards >}}
