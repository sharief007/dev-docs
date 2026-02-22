# L1: Networking

> Networking is the connective tissue of every distributed system. Know each layer deeply.

---

## 1. OSI Model — All 7 Layers

```
Layer 7 — Application    HTTP, DNS, SMTP, FTP, gRPC
Layer 6 — Presentation   TLS/SSL, encoding, compression
Layer 5 — Session        Session establishment, checkpointing
Layer 4 — Transport      TCP, UDP, QUIC (ports, segmentation)
Layer 3 — Network        IP, ICMP, routing (logical addressing)
Layer 2 — Data Link      Ethernet, MAC, ARP, switches, VLANs
Layer 1 — Physical       Cables, fiber, radio, signal encoding
```

Each layer only communicates with adjacent layers. Headers are added/stripped at each layer (encapsulation/decapsulation).

---

## 2. Physical & Data Link Layer

### Physical
- Copper (Cat5e/6/6a), fiber (single-mode vs multi-mode), wireless (802.11 variants)
- Signal encoding: NRZ, Manchester encoding, PAM4 (for 400GbE)
- Bandwidth vs throughput vs latency distinction

### Data Link
- **MAC addresses**: 48-bit hardware address (locally unique)
- **ARP (Address Resolution Protocol)**: resolves IP → MAC within a subnet
- **Ethernet frames**: preamble, MAC dst, MAC src, EtherType, payload, FCS
- **Switches**: operate at L2, learn MAC table, flood unknown destinations
- **VLANs (802.1Q)**: logical network segmentation on shared physical infrastructure
- **Spanning Tree Protocol (STP)**: prevents broadcast storms in switched networks
- **LACP (Link Aggregation)**: bond multiple physical links for redundancy/bandwidth

---

## 3. Network Layer (IP)

### IPv4
- 32-bit addresses, CIDR notation (e.g., 10.0.0.0/16)
- Subnetting: host bits vs network bits
- Private address ranges: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- IP fragmentation (MTU = 1500 bytes for Ethernet)
- TTL (Time To Live): decremented at each hop, prevents routing loops

### IPv6
- 128-bit addresses, eliminates NAT need
- Stateless Address Autoconfiguration (SLAAC)
- No fragmentation by routers (must do it at source)
- Mandatory IPSec support

### Routing
- **Static routes**: manually configured, simple but not scalable
- **Dynamic routing protocols**:
  - **OSPF** (Open Shortest Path First): link-state, Dijkstra-based, within an AS
  - **BGP** (Border Gateway Protocol): path-vector, between autonomous systems, runs the Internet
  - **RIP**: distance-vector, hop count metric, obsolete for large networks
- **Autonomous Systems (AS)**: logical groupings of IP networks under one organization
- **Anycast**: same IP advertised from multiple locations, traffic goes to nearest

### NAT (Network Address Translation)
- Enables many private IPs to share one public IP
- Port address translation (PAT / NAPT): tracks connections by {src IP, src port, dst IP, dst port}
- NAT breaks true end-to-end connectivity (problem for P2P, WebRTC)
- STUN/TURN/ICE solve NAT traversal for WebRTC

### ICMP
- Used by ping, traceroute
- Error reporting (destination unreachable, time exceeded, redirect)
- Path MTU Discovery (PMTUD)

---

## 4. Transport Layer

### TCP — Transmission Control Protocol

**Connection Management**
- Three-way handshake: SYN → SYN-ACK → ACK (1 RTT overhead)
- Four-way termination: FIN → ACK, FIN → ACK
- TIME_WAIT state: 2×MSL (up to 4 minutes), prevents port reuse
  - Optimization: SO_REUSEADDR, SO_LINGER

**Reliable Delivery**
- Sequence numbers and acknowledgments
- Cumulative ACK vs selective ACK (SACK)
- Retransmission timeout (RTO) — exponential backoff
- Fast retransmit: 3 duplicate ACKs triggers retransmit before timeout

**Flow Control**
- Receive window (rwnd) — receiver tells sender how much buffer it has
- Zero window probing when rwnd=0
- Window scaling option (for high-BDP links)

**Congestion Control**
- Slow start: exponential growth up to ssthresh
- Congestion avoidance: linear growth (AIMD — additive increase multiplicative decrease)
- **TCP RENO**: on loss → halve cwnd
- **TCP CUBIC**: default Linux algorithm, cubic window growth (better for high-BDP)
- **TCP BBR** (Bottleneck Bandwidth and Round-trip propagation time): model-based, Google's algorithm, doesn't use loss as signal — better for WAN
- **TCP Vegas**: delay-based (less aggressive, better for short RTT)

**Nagle Algorithm**
- Coalesces small packets to reduce overhead
- Disable with TCP_NODELAY for latency-sensitive applications

**TCP Head-of-Line Blocking**
- All streams share one connection — one lost packet blocks all
- Solved by HTTP/2 multiplexing? No — HOL blocking moves to TCP level
- Truly solved by QUIC (UDP-based)

**TCP Offloading**
- TSO (TCP Segmentation Offload): NIC segments large buffers
- GRO (Generic Receive Offload): NIC coalesces small packets on receive
- Checksum offload

### UDP — User Datagram Protocol

- No connection establishment (no 3-way handshake overhead)
- No reliability, no ordering guarantees
- No flow/congestion control
- **When to use UDP**: DNS, NTP, video streaming, games, WebRTC (where you want to control reliability)
- UDP multicast: one sender → many receivers (used in financial data broadcasting)

### QUIC Protocol

> HTTP/3 is built on QUIC. The future of web transport.

- Built on UDP, implements reliability/ordering per-stream in userspace
- **0-RTT connection establishment** (TLS 1.3 session resumption baked in)
- **Stream multiplexing**: each stream is independent — losing a packet in stream 1 doesn't block stream 2
- **Connection migration**: connection ID is not tied to IP:port, survives mobile handoffs
- **Forward Error Correction (FEC)**: optional recovery without retransmit
- **Built-in encryption**: QUIC mandates TLS 1.3 — no unencrypted QUIC
- Deployment complexity: middleboxes block UDP on port 443; Google uses port 443

---

## 5. Application Layer Protocols

### DNS — Domain Name System

**Resolution Chain**
```
Browser → OS resolver → Recursive resolver (ISP/8.8.8.8)
    → Root servers (.com, .org) → TLD nameservers → Authoritative nameserver
```

**Record Types**
| Record | Purpose | Example |
|--------|---------|---------|
| A | IPv4 address | example.com → 93.184.216.34 |
| AAAA | IPv6 address | example.com → 2606:2800::1 |
| CNAME | Alias to another name | www → example.com |
| MX | Mail server | priority + hostname |
| TXT | Arbitrary text | SPF, DKIM, domain verification |
| NS | Authoritative nameserver | |
| SOA | Zone metadata | serial, refresh, retry, expire |
| SRV | Service location | _http._tcp.example.com |
| PTR | Reverse lookup (IP → name) | |
| CAA | Restrict certificate authorities | |

**DNS Caching & TTL**
- TTL controls caching duration — low TTL = faster propagation, more DNS load
- Negative caching (NXDOMAIN) — SOA minimum TTL

**DNSSEC**
- Cryptographic signatures on DNS records
- Chain of trust from root
- Prevents DNS spoofing / cache poisoning

**DNS over HTTPS (DoH) and DNS over TLS (DoT)**
- Prevents ISP snooping on DNS queries
- Used by Firefox (Cloudflare 1.1.1.1 by default)

**GeoDNS**
- Returns different A records based on client geography
- Used for: traffic steering to nearest data center
- Limitation: client IP may be resolver IP (not actual user) → use EDNS Client Subnet

### HTTP/1.1

- Text-based protocol
- Methods: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS
- Status codes: 1xx (info), 2xx (success), 3xx (redirect), 4xx (client error), 5xx (server error)
- **Persistent connections** (Connection: keep-alive): reuse TCP connections
- **Pipelining**: send multiple requests without waiting for responses — disabled by default (HOL blocking)
- Chunked transfer encoding: stream response without known Content-Length
- Caching headers: Cache-Control, ETag, Last-Modified, Vary

### HTTP/2

- **Binary framing layer**: no more text parsing, multiplexed frames
- **Header compression (HPACK)**: static + dynamic header table, Huffman encoding
- **Stream multiplexing**: multiple requests over one TCP connection, each with stream ID
- **Server push**: server sends resources before client asks (rarely useful in practice)
- **HTTP/2 HOL blocking**: still exists at TCP level — one lost packet blocks all streams
- **Stream priorities and flow control** at application level

### HTTP/3 (QUIC-based)

- Replaces TCP with QUIC at transport layer
- Eliminates TCP HOL blocking — each HTTP/3 stream is an independent QUIC stream
- 0-RTT connection establishment
- Connection migration (switch from WiFi to 4G without reconnect)
- Mandatory TLS 1.3

### TLS/SSL

**TLS 1.2 Handshake (2 RTTs)**
```
Client → Server: ClientHello (cipher suites, random)
Server → Client: ServerHello, Certificate, ServerHelloDone
Client → Server: ClientKeyExchange, ChangeCipherSpec, Finished
Server → Client: ChangeCipherSpec, Finished
```

**TLS 1.3 Handshake (1 RTT, 0-RTT resumption)**
- Only 5 cipher suites (all with PFS)
- No RSA key exchange (all use ephemeral Diffie-Hellman)
- 0-RTT early data (replay attack risk)

**Key Concepts**
- **PKI (Public Key Infrastructure)**: CAs issue certificates
- **Certificate chain**: leaf cert → intermediate CA → root CA
- **Certificate Transparency (CT)**: public log of issued certs
- **OCSP stapling**: server provides cert revocation status
- **Perfect Forward Secrecy (PFS)**: ephemeral key exchange — past sessions safe if private key leaked
- **SNI (Server Name Indication)**: hostname in TLS handshake (needed for shared IP hosting)
- **mTLS (mutual TLS)**: both client and server authenticate (used in service meshes)

### WebSockets

- **Upgrade handshake**: HTTP GET with `Upgrade: websocket` → 101 Switching Protocols
- **Full-duplex** over a single TCP connection
- **Frame structure**: FIN bit, opcode, masking, payload length, masking key, payload
- Message types: text, binary, ping, pong, close
- **vs SSE**: SSE is server→client only, HTTP-based, auto-reconnect
- **vs Long Polling**: HTTP request held open until data available, then new request
- **Scaling WebSockets**: sticky sessions or Redis pub/sub for cross-node message delivery

### WebRTC

**Architecture**
- P2P: direct browser-to-browser (requires NAT traversal)
- SFU (Selective Forwarding Unit): server receives all streams, forwards selectively — scales to large groups
- MCU (Multipoint Control Unit): server mixes streams into one — CPU heavy, low bandwidth for clients
- **SFU vs MCU**: SFU preferred (Zoom, Teams, Google Meet use SFU)

**NAT Traversal (ICE)**
- **STUN** (Session Traversal Utilities for NAT): discover your public IP and port
- **TURN** (Traversal Using Relays around NAT): relay server when direct P2P fails
- **ICE** (Interactive Connectivity Establishment): tries direct → STUN → TURN in order
- Trickle ICE: stream candidates as they're discovered (reduces latency)

**Signaling**
- WebRTC doesn't define signaling — you implement it (WebSocket, HTTP, etc.)
- **SDP (Session Description Protocol)**: describes media capabilities (codecs, bitrate, etc.)
- Offer/answer model: one peer sends offer SDP, other responds with answer SDP

**Security**
- **DTLS** (Datagram TLS): TLS over UDP — encrypts QUIC transport
- **SRTP** (Secure Real-Time Protocol): encrypts media streams
- **SRTCP**: encrypts RTCP control channel

### gRPC

- Built on HTTP/2 (streams, multiplexing)
- Uses Protocol Buffers (binary, typed, backward-compatible)
- **4 service types**:
  - Unary: one request → one response
  - Server streaming: one request → stream of responses
  - Client streaming: stream of requests → one response
  - Bidirectional streaming: stream ↔ stream
- **Advantages over REST**: strong typing, code generation, HTTP/2 performance, streaming
- **Disadvantages**: not human-readable, harder to debug, browser support via grpc-web proxy
- **vs GraphQL**: gRPC is RPC-style, GraphQL is query-style (better for flexible clients)

### REST Design
- Richardson Maturity Model (Level 0 → 3: Hypermedia)
- Idempotent methods (GET, PUT, DELETE) vs non-idempotent (POST)
- Stateless server (all state in request)
- HATEOAS (Hypermedia as the Engine of Application State)

### GraphQL
- Single endpoint, client specifies exact fields
- N+1 query problem and DataLoader (batching solution)
- Schema introspection
- Mutations, subscriptions (over WebSocket)

---

## 6. Load Balancing

### L4 vs L7 Load Balancing
| | L4 (Transport) | L7 (Application) |
|--|--|--|
| Works on | IP + TCP/UDP | HTTP headers, cookies, body |
| Performance | Faster (no decryption) | Slower but smarter |
| TLS termination | No | Yes |
| Content-based routing | No | Yes |
| Example | AWS NLB, LVS | NGINX, HAProxy, AWS ALB |

### Load Balancing Algorithms
- **Round Robin**: equal distribution, ignores server capacity
- **Weighted Round Robin**: accounts for server capacity
- **Least Connections**: routes to server with fewest active connections
- **Weighted Least Connections**: capacity-aware least connections
- **IP Hash**: consistent routing based on source IP (sticky without cookie)
- **Consistent Hashing**: minimal redistribution when servers added/removed
- **Random with Two Choices (P2C)**: pick 2 random, route to least loaded — power of two choices beats pure random

### Health Checks
- Passive: detect failures from errors in real traffic
- Active: periodic synthetic probes (HTTP 200, TCP connect, custom script)
- Health check intervals, thresholds for marking up/down

### Global Server Load Balancing (GSLB)
- Distributes across geographic regions
- Methods: GeoDNS, Anycast BGP
- Handles regional failover

### Anycast Routing
- Same IP prefix advertised from multiple locations via BGP
- Traffic naturally routes to topologically closest instance
- Used by: Cloudflare, Google DNS (8.8.8.8), CDN edge nodes

---

## 7. CDN — Content Delivery Networks

### Architecture
- **Origin server**: your actual servers
- **Edge nodes / PoPs (Points of Presence)**: globally distributed caches
- **Cache fill**: edge pulls from origin on cache miss (origin pull) or you push content (CDN push)

### Caching at the Edge
- Cache keys: URL, headers (Accept-Encoding, Accept), query params
- Cache-Control directives: max-age, s-maxage, no-cache, no-store, stale-while-revalidate
- ETags and conditional requests (304 Not Modified)
- Vary header: cache separate copies based on header values

### Cache Invalidation
- TTL expiry (eventually consistent)
- Instant purge by URL, path, or tag
- Surrogate keys / cache tags (Fastly, Cloudflare)

### Dynamic Content Acceleration
- TCP connection reuse to origin (keep persistent TCP connections over WAN)
- Anycast routing optimizes last-mile to edge
- Protocol optimization (HTTP/2 to origin even if client uses HTTP/1.1)
- Edge-side includes (ESI) for partial page caching

### Edge Computing
- **Lambda@Edge** (AWS): run Node.js/Python at CloudFront edges
- **Cloudflare Workers**: V8 isolates, 0-ms cold start
- **Fastly Compute@Edge**: WASM-based
- Use cases: A/B testing at edge, auth at edge, request rewriting, geofencing

---

## 8. Network Security

### Firewalls
- **Stateless**: filter packets based on IP/port rules only
- **Stateful**: tracks connection state (allows established TCP flows)
- **NGFW** (Next-Gen Firewall): deep packet inspection, application awareness

### DDoS Mitigation
- **Volumetric attacks**: UDP flood, ICMP flood — absorb at ISP/Anycast level
- **Protocol attacks**: SYN flood — SYN cookies, rate limiting
- **Application attacks**: HTTP flood, Slowloris — L7 detection, bot fingerprinting
- Cloudflare, AWS Shield, Akamai WAF/DDoS services

### VPN Protocols
- **IPSec**: L3 tunnel, complex, interoperable
- **OpenVPN**: SSL-based, flexible
- **WireGuard**: modern, simple, fast (uses Noise protocol + ChaCha20)

### Zero Trust Networking
- "Never trust, always verify" — even internal traffic is authenticated
- Identity-based access (not network location)
- Micro-segmentation
- BeyondCorp (Google's implementation), Cloudflare Access, Zscaler

### Service Mesh Security
- mTLS between all services (Istio, Linkerd)
- Certificate rotation, automated via cert-manager
- Authorization policies (service-to-service allow/deny)
