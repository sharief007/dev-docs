---
title: TCP vs UDP
weight: 1
type: docs
---

A user complains your video call has a 400ms lag spike every time their WiFi drops a packet. You inspect the protocol — it's running over TCP, which retransmits the lost packet and stalls every byte behind it until recovery. Switch the call to UDP and the lost frame is just gone; the next frame plays immediately. Two seconds later a different user reports their database query came back with corrupted bytes — turns out that one was incorrectly running over UDP. The choice between TCP and UDP is the most foundational protocol decision in any system, and HTTP/3, QUIC, video streaming, and gaming exist precisely because the answer isn't always TCP.

TCP and UDP are the two transport-layer protocols that almost everything on the internet runs on. They sit above IP and below application protocols like HTTP, DNS, and WebSocket.

| | TCP | UDP |
|---|---|---|
| Connection | Stateful (3-way handshake) | Connectionless |
| Delivery guarantee | Yes (retransmit on loss) | No |
| Ordering | Yes (sequence numbers) | No |
| Flow control | Yes (receive window) | No |
| Congestion control | Yes (slow start, AIMD) | No |
| Header size | 20 bytes minimum | 8 bytes |
| Latency | Higher (handshake + reliability overhead) | Lower |
| Use cases | HTTP, database, file transfer, SSH | DNS, video streaming, gaming, VoIP, QUIC |

## TCP

### 3-Way Handshake

Every TCP connection starts with a 3-way handshake before any application data is exchanged.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: SYN (seq=x)
    S->>C: SYN-ACK (seq=y, ack=x+1)
    C->>S: ACK (ack=y+1)
    Note over C,S: Connection established — 1 RTT
    C->>S: HTTP Request (application data)
```

Cost: **1 RTT before the first byte of application data**. TLS adds an additional 1 RTT on top (TLS 1.3), making the total **2 RTTs** for a new HTTPS connection.

The handshake allocates state on both sides (socket buffers, sequence numbers, congestion window). This is why half-open connections and SYN floods are effective attacks — they exhaust this state.

### Reliable Ordered Delivery

- Every byte has a **sequence number**
- Receiver sends **ACK** for received data; ACK number = next expected byte
- Sender starts a **retransmission timer** on each segment; retransmits if ACK not received before timeout
- Receiver **buffers out-of-order segments** and delivers to the application only in order

```
Sender:   [1][2][3][4][5]
Network:      ↑ packet 2 lost
Receiver: [1]   [3][4][5]  ← buffers 3,4,5; delivers only 1 to app
Receiver sends: ACK 2 (NACK, requesting retransmit)
Sender retransmits: [2]
Receiver: [1][2][3][4][5]  ← delivers 2,3,4,5 to app
```

This in-order delivery requirement is the root cause of **TCP-level head-of-line blocking** (covered in HTTP/2 and HTTP/3).

### Flow Control — Receive Window

The receiver advertises how much buffer space it has. The sender cannot have more than `rwnd` bytes of unacknowledged data in flight.

```
Receiver buffer: [  consumed  |    available (rwnd=4KB)   ]
Sender: may only send 4KB before receiving an ACK
```

If the application is reading slowly, `rwnd` shrinks → sender slows down → no buffer overflow at the receiver.

**Window scaling:** The original TCP window field is 16 bits (max 65KB). For high-bandwidth or high-latency links (e.g., 10Gbps WAN), 65KB in flight is insufficient. The Window Scale option (negotiated during handshake) multiplies the window by up to 2¹⁴ — enabling windows up to 1GB.

### Congestion Control

TCP infers network congestion from packet loss and adjusts the send rate. The **congestion window** (`cwnd`) limits how much the sender sends, independent of `rwnd`.

Effective send rate = `min(cwnd, rwnd)`

{{< tabs items="Slow Start,Congestion Avoidance (AIMD),Fast Retransmit" >}}
  {{< tab >}}
  Starts cautiously to probe available bandwidth.

  - Initial `cwnd` = 10 MSS (Maximum Segment Size, ~14KB)
  - `cwnd` **doubles** every RTT (exponential growth)
  - Continues until `cwnd` reaches `ssthresh` (slow start threshold)
  - On loss: `ssthresh = cwnd/2`, restart from initial `cwnd`

  New connections always go through slow start — this is why the first few seconds of a large transfer are slow. HTTP/2's single connection amplifies this: one connection means one slow start.
  {{< /tab >}}

  {{< tab >}}
  Once past `ssthresh`, TCP shifts to linear growth.

  - **Additive Increase**: `cwnd += 1 MSS` per RTT
  - **Multiplicative Decrease**: on loss, `cwnd = cwnd / 2`
  - Net effect: "probe up, back off hard" — the sawtooth pattern

  ```
  cwnd
    ↑       /\      /\      /\
    |      /  \    /  \    /  \
    |     /    \  /    \  /    \
    |____/      \/      \/      \___
    ssthresh        time →
  ```

  AIMD ensures fairness between competing flows — all flows converge to equal share of bottleneck bandwidth.
  {{< /tab >}}

  {{< tab >}}
  Avoids waiting for retransmission timeout (which can be 200ms–1s).

  - Receiver sends **duplicate ACKs** for each out-of-order segment received
  - **3 duplicate ACKs** → sender retransmits the missing segment immediately, without waiting for timeout
  - Combined with **Fast Recovery**: `ssthresh = cwnd/2`, `cwnd = ssthresh` (not full restart)

  Fast retransmit + recovery recovers from isolated packet loss quickly. Timeout-based retransmit is only triggered for more severe loss or when duplicate ACKs don't arrive.
  {{< /tab >}}
{{< /tabs >}}

{{< callout type="info" >}}
**BBR (Bottleneck Bandwidth and Round-trip propagation time):** Google's 2016 congestion control algorithm. Instead of reacting to loss, BBR models the network's bottleneck bandwidth and minimum RTT, and sends at the estimated optimal rate. More aggressive than AIMD on long-fat pipes; used by Google, YouTube, and many CDNs.
{{< /callout >}}

### Connection Teardown — TIME_WAIT

TCP uses a 4-way teardown. Either side initiates by sending FIN.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: FIN (active close)
    S->>C: ACK
    Note over S: Server can still send data (half-close)
    S->>C: FIN
    C->>S: ACK
    Note over C: TIME_WAIT starts (2 × MSL)
```

The active closer enters **TIME_WAIT** for `2 × MSL` (Maximum Segment Lifetime). Linux pins this to a fixed 60 seconds via the `TCP_TIMEWAIT_LEN` constant rather than computing it from MSL; BSD-derived stacks typically use 60–120 seconds.

**Why TIME_WAIT exists:**
1. Ensures the final ACK reaches the server (if lost, server retransmits FIN; client must be alive to re-ACK)
2. Prevents delayed packets from an old connection from being misinterpreted by a new connection on the same src:dst IP:port pair

**TIME_WAIT at scale:**

At high connection rates (100K+ connections/second), sockets pile up in TIME_WAIT. Each socket holds a port. The default local port range is ~28,000 ports → exhausted quickly.

| Mitigation | How |
|------------|-----|
| `net.ipv4.tcp_tw_reuse = 1` | Reuse TIME_WAIT sockets for new outbound connections if safe (requires timestamps) |
| `net.ipv4.ip_local_port_range = 1024 65535` | Increase available local ports from ~28K to ~64K |
| **Connection pooling** | Reuse TCP connections instead of closing after each request — eliminates most TIME_WAIT accumulation |
| `SO_REUSEPORT` | Multiple sockets can bind the same port; load distributed across sockets |

{{< callout type="warning" >}}
`tcp_tw_recycle` (older Linux) aggressively recycled TIME_WAIT sockets but broke connections from clients behind NAT — multiple clients share one IP and their timestamps were non-monotonic from the server's perspective. It was removed in Linux 4.12. Do not use it.
{{< /callout >}}

### Half-Open Connections

A half-open connection exists when one side believes the connection is established but the other does not (crash, NAT timeout, network partition).

- NAT devices drop mappings after idle timeout (typically 30s–5 minutes). The client still holds the socket; the NAT has forgotten the mapping.
- The remote side sends data → NAT has no mapping → returns RST → connection reset
- Without application-level data flow, half-open connections can persist indefinitely

**Detection:**
- **TCP keepalive**: kernel sends probe packets after idle period (`tcp_keepalive_time`, default 2 hours — far too long for most applications)
- **Application heartbeat**: preferred over TCP keepalive; more control, works across proxies that strip TCP options

### SYN Flood

An attacker sends SYN packets without completing the handshake. The server allocates state for each half-open connection in the **SYN backlog**. When the backlog is full, legitimate SYNs are dropped.

**SYN cookies** (default on Linux):
- Server encodes connection state into the Initial Sequence Number (ISN) of the SYN-ACK
- No memory allocated until the client's ACK arrives
- ACK carries the encoded state — server reconstructs the connection from it
- Attackers never send ACK → no memory consumed

## UDP

UDP provides only two things beyond raw IP: port numbers (multiplexing) and an optional checksum.

```
UDP Header (8 bytes):
┌──────────────┬──────────────┐
│  Src Port    │  Dst Port    │
├──────────────┼──────────────┤
│  Length      │  Checksum    │
└──────────────┴──────────────┘
│  Payload (application data) │
```

Each UDP datagram is **independent**. There is no connection state, no retransmission, no ordering, no flow control. If a packet is lost, it is gone.

**What UDP gives you:**
- No handshake latency — send immediately
- No head-of-line blocking — each datagram is independent
- Multicast and broadcast support (TCP is unicast only)
- Application controls retry logic (if needed)

## Why QUIC Chose UDP

TCP is implemented in the OS kernel. Deploying a change to TCP behavior requires an OS update across all devices — a multi-year rollout. The internet had been unable to evolve TCP for decades due to this.

QUIC runs in user space (part of the application binary). Updating QUIC behavior requires only an application update. By building on UDP, QUIC:
- Bypasses kernel TCP entirely
- Reimplements reliability, ordering, and flow control — per stream, independently
- Adds 0-RTT resumption, connection migration, and built-in TLS 1.3
- Can be deployed and iterated on at app-update speed

See [HTTP/3 and QUIC](../http-3) for the full treatment.

## Use Cases

| Protocol | Use | Reason |
|----------|-----|--------|
| TCP | HTTP/1.1, HTTP/2 | Reliability and ordering required |
| TCP | Database queries (PostgreSQL, MySQL) | Results must be complete and ordered |
| TCP | File transfer (SFTP, rsync) | Every byte must arrive |
| TCP | SSH | Interactive; cannot lose keystrokes |
| UDP | DNS | Single request/response; latency matters more than reliability; retry at app layer |
| UDP | Video streaming (RTP/RTSP) | Occasional loss acceptable; retransmitting old frames wastes bandwidth |
| UDP | Online gaming | Old positional updates are useless; prefer freshest data over completeness |
| UDP | VoIP | Real-time; old audio frames are worthless; retransmit would arrive too late |
| UDP | QUIC (HTTP/3) | Reliability reimplemented in user space with independent streams |
| UDP | DHCP, NTP, SNMP | Simple request/response; broadcast support needed |

{{< callout type="info" >}}
**Interview tip:** When asked TCP vs UDP, frame the choice on what you can tolerate losing: "I'd default to TCP for anything where every byte must arrive in order — HTTP, databases, file transfer. The cost is the 1-RTT handshake (2 RTT with TLS 1.3) and TCP-level HOL blocking. I'd use UDP when freshness beats completeness — VoIP, live video, multiplayer games — because retransmitting a 50ms-old audio frame is worse than dropping it. At scale I'd watch for TIME_WAIT exhaustion eating local ports — fix with connection pooling and `tcp_tw_reuse`, never the broken `tcp_tw_recycle`. And HTTP/3 chose UDP so QUIC could reimplement reliability per-stream in user space, sidestepping kernel TCP entirely."
{{< /callout >}}

## Test Your Understanding

{{< details title="HTTP/2 multiplexes many streams over a single TCP connection. A single packet is lost. How many streams are affected, and why is this worse than HTTP/1.1?" closed="true" >}}
**All streams are affected.** TCP guarantees in-order delivery, so a lost packet blocks the entire receive buffer until retransmission completes. Every HTTP/2 stream sharing that connection stalls — even streams whose data arrived fine.

HTTP/1.1 uses 6 parallel TCP connections, so a lost packet on one connection blocks only ~1/6 of requests. HTTP/2's single-connection design makes TCP-level HOL blocking **worse**, not better. This is the core reason HTTP/3 uses QUIC — each QUIC stream has independent loss recovery.
{{< /details >}}

{{< details title="A high-RPS service creates 100K short-lived TCP connections per second. After a few minutes, new connections start failing. What's happening and what are three ways to fix it?" closed="true" >}}
**TIME_WAIT exhaustion.** Each closed connection enters TIME_WAIT for 60 seconds (Linux). At 100K connections/sec, that's 6 million sockets in TIME_WAIT, each holding a local port. The default port range (~28K ports) is exhausted in under a second.

**Fixes:**
1. **Connection pooling** — reuse TCP connections instead of closing them. Eliminates most TIME_WAIT accumulation. This is the correct architectural fix.
2. **`net.ipv4.tcp_tw_reuse=1`** — allows reusing TIME_WAIT sockets for new outbound connections (requires TCP timestamps enabled).
3. **Expand port range** — `net.ipv4.ip_local_port_range = 1024 65535` increases from ~28K to ~64K available ports. A band-aid, not a fix.

Note: `tcp_tw_recycle` was removed in Linux 4.12 because it broke connections from clients behind NAT (non-monotonic timestamps from shared IPs).
{{< /details >}}

{{< details title="You're designing a multiplayer game server. You choose UDP for game state updates. A player reports that their character sometimes teleports backward. What's happening?" closed="true" >}}
**Out-of-order delivery.** UDP provides no ordering guarantees. If packet 5 (position: x=100) arrives after packet 6 (position: x=110), the client applies them in arrival order — the character jumps to x=110, then snaps back to x=100.

**Fix:** Include a **sequence number** in each UDP packet. The client discards any packet with a sequence number ≤ the last applied one. This gives you "latest-wins" semantics without TCP's retransmission overhead. For state that must be reliable (inventory changes, score updates), use a separate TCP connection or implement selective ACK/retransmit in the application protocol.
{{< /details >}}

{{< details title="TCP's congestion control uses slow start. A new connection starts with cwnd=10 MSS (~14KB). Why does this hurt HTTP/2 more than HTTP/1.1 in the first few hundred milliseconds?" closed="true" >}}
HTTP/2 uses **one TCP connection** per origin. That single connection starts with a 14KB congestion window. All multiplexed streams share this tiny initial bandwidth — if a page needs 200KB of resources, slow start takes several RTTs to ramp up.

HTTP/1.1 opens **6 parallel connections**, each with its own slow start. 6 × 14KB = 84KB of initial burst capacity — 6x more data can flow in the first RTT. Each connection ramps up independently.

**Mitigations:** Increase the initial cwnd (Linux: `ip route change ... initcwnd 20`), use TCP Fast Open (TFO) to send data in the SYN packet, or enable connection prewarming for critical paths.
{{< /details >}}

{{< details title="SYN cookies defend against SYN floods by encoding connection state in the ISN. What information is lost when using SYN cookies, and what's the practical impact?" closed="true" >}}
SYN cookies encode connection parameters (MSS, timestamp, window scale) into the ISN, so the server allocates **no memory** until the ACK arrives. But the ISN has limited bits, so some options can't be preserved:

1. **TCP window scaling** — may be lost or limited. Without it, the receive window is capped at 64KB — severely limiting throughput on high-bandwidth or high-latency links.
2. **Selective ACK (SACK)** — may not be negotiated. Without SACK, the sender must retransmit all data from a lost packet onward, not just the missing segments.
3. **TCP timestamps** — may be omitted. Timestamps are used for RTT estimation and PAWS (Protection Against Wrapped Sequence numbers).

**Practical impact:** Under normal traffic (no attack), SYN cookies are not used and full TCP options are negotiated. During a SYN flood, connections established via SYN cookies may have degraded performance (no window scaling, no SACK) — but they at least succeed rather than being dropped. It's a graceful degradation, not a full defense.
{{< /details >}}

{{< details title="A service runs behind a NAT gateway. After 5 minutes of inactivity, clients report 'connection reset' errors. TCP keepalive is set to the default 2 hours. What's happening?" closed="true" >}}
The **NAT gateway** drops idle connection mappings after its timeout (typically 30s–5 minutes, depending on the cloud provider or hardware). The client and server still hold their TCP sockets, but the NAT has forgotten the mapping.

When either side sends data, the NAT has no entry for the 5-tuple → it returns a **RST** (reset), killing the connection. TCP keepalive at 2 hours is far too infrequent to prevent this.

**Fix:** Lower TCP keepalive to less than the NAT timeout (e.g., `tcp_keepalive_time=60, tcp_keepalive_intvl=10, tcp_keepalive_probes=3`). Or better: use **application-level heartbeats** (e.g., gRPC keepalive pings, WebSocket ping/pong frames) which work across proxies that strip TCP options. AWS NAT Gateway timeout is 350 seconds for TCP — set keepalive below that.
{{< /details >}}
