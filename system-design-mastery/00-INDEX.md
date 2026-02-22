# System Design Mastery — Complete Knowledge Hierarchy

> A comprehensive, ordered hierarchy of every concept you need to master system design at any scale.
> Topics are ordered by dependency: learn each level before moving to the next.

---

## How to Use This Roadmap

Each section links to a detailed file. Work through levels **sequentially** — later levels build directly on earlier ones. The tree format shows **what depends on what**.

---

## Master Hierarchy Tree

```
SYSTEM DESIGN MASTERY
│
├── L0: CS & Math Foundations ──────────────────── [01-cs-and-math-foundations.md]
│   ├── Discrete Math (logic, sets, combinatorics, graph theory)
│   ├── Linear Algebra (vectors, matrices, SVD, PCA)
│   ├── Probability & Statistics (Bayes, distributions, A/B testing)
│   ├── Complexity Theory (Big O, NP-completeness, amortized analysis)
│   ├── Data Structures (arrays → tries → segment trees → CRDTs)
│   ├── Algorithms (sorting → graph → string → DP)
│   ├── Computer Architecture (CPU pipeline, memory hierarchy, NUMA, GPU)
│   └── OS Internals (scheduling, synchronization, VFS, io_uring, epoll)
│
├── L1: Networking ──────────────────────────────── [02-networking.md]
│   ├── OSI Model (all 7 layers internals)
│   ├── TCP/IP Stack (3-way handshake, congestion control, QUIC)
│   ├── DNS (resolution, record types, DNSSEC, GeoDNS)
│   ├── HTTP/1.1 → HTTP/2 → HTTP/3 (internals, HOL blocking)
│   ├── TLS/SSL (handshake, PKI, PFS, TLS 1.3)
│   ├── WebSockets, SSE, WebRTC (ICE, STUN, TURN, SFU vs MCU)
│   ├── gRPC, REST, GraphQL (tradeoffs, Protocol Buffers)
│   ├── Load Balancing (L4 vs L7, algorithms, GSLB, Anycast)
│   └── CDN (edge caching, invalidation, Lambda@Edge)
│
├── L2: Storage & Databases ────────────────────── [03-storage-and-databases.md]
│   ├── Storage Hardware (HDD vs SSD, NVMe, RAID, object storage)
│   ├── File Systems (inodes, journaling, CoW, distributed FS)
│   ├── RDBMS Internals (ACID, B-trees, MVCC, WAL, query plans)
│   ├── NoSQL: Key-Value, Document, Wide-Column, Graph, Time-Series
│   ├── Search Engines (inverted index, BM25, Lucene segments)
│   ├── Vector Databases (HNSW, IVF, PQ, ANN algorithms)
│   ├── Storage Algorithms (LSM trees, B-trees tradeoffs, write amplification)
│   ├── Caching (LRU/LFU/ARC, stampede prevention, layered caching)
│   └── Distributed DB Concepts (sharding, consistent hashing, replication)
│
├── L3: Distributed Systems Theory ─────────────── [04-distributed-systems-theory.md]
│   ├── Theoretical Limits (CAP, PACELC, FLP, Two Generals, Byzantine)
│   ├── Consistency Models (linearizability → causal → eventual)
│   ├── Time & Ordering (Lamport clocks, vector clocks, HLC, TrueTime)
│   ├── Consensus (Paxos, Raft, ZAB, PBFT, HotStuff)
│   ├── CRDTs (G-Counter, OR-Set, RGA, Yjs, Automerge)
│   ├── Distributed Hash Tables (Chord, Kademlia)
│   ├── Fault Tolerance (quorums, gossip, phi accrual, STONITH)
│   ├── Coordination (ZooKeeper, etcd, Consul, distributed locks)
│   └── Distributed Transactions (2PC, 3PC, Saga, TCC, outbox)
│
├── L4: Infrastructure & DevOps ────────────────── [05-infrastructure-and-devops.md]
│   ├── Containerization (Docker internals, namespaces, cgroups)
│   ├── Kubernetes (architecture, workloads, networking, operators)
│   ├── Service Mesh (Istio, Linkerd, eBPF/Cilium)
│   ├── Cloud Architecture (AWS/GCP/Azure, multi-region, DR)
│   ├── Infrastructure as Code (Terraform, Pulumi, Ansible)
│   ├── CI/CD (blue-green, canary, rolling, GitOps)
│   └── Observability (logs, metrics, traces, SLO, chaos engineering)
│
├── L5: Architecture Patterns ──────────────────── [06-architecture-patterns.md]
│   ├── Estimation (latency numbers, capacity planning methodology)
│   ├── API Design (REST maturity, GraphQL, gRPC, pagination, idempotency)
│   ├── Microservices (DDD, bounded contexts, service decomposition)
│   ├── Event-Driven (event sourcing, CQRS, saga, outbox, DLQ)
│   ├── Messaging Systems (Kafka deep dive, RabbitMQ, Pulsar, NATS)
│   ├── Serverless (Lambda, cold start, Step Functions)
│   └── Reliability Patterns (circuit breaker, bulkhead, retry, rate limiting)
│
├── L6: Specialized Systems ────────────────────── [07-specialized-systems.md]
│   ├── Search Systems (inverted index, suffix trees, Aho-Corasick, Elasticsearch)
│   ├── Geospatial (GeoHash, QuadTree, R-tree, H3, S2, route optimization)
│   ├── Real-Time Communication (WebSocket scale, WebRTC, MQTT, presence)
│   ├── Media Delivery (HLS/DASH, adaptive bitrate, live streaming)
│   ├── Notification Systems (APNs/FCM, fanout strategies)
│   ├── Payment Systems (idempotency, ledgers, PCI DSS)
│   └── Scheduling & Workflows (job queues, Temporal, Airflow, fair scheduling)
│
├── L7: Data Engineering ───────────────────────── [08-data-engineering.md]
│   ├── Paradigms (Lambda, Kappa, Delta, Data Mesh)
│   ├── Batch Processing (Hadoop, Spark internals, Presto/Trino)
│   ├── Stream Processing (Flink deep dive, watermarks, exactly-once)
│   ├── Data Warehouses (columnar storage, star schema, Snowflake, BigQuery)
│   ├── Data Lakes (Delta Lake, Iceberg, Hudi, data catalogs)
│   ├── ETL/ELT (Airflow, dbt, CDC with Debezium, schema evolution)
│   └── Feature Engineering (feature stores, point-in-time correctness)
│
├── L8: AI & ML Systems ────────────────────────── [09-ai-ml-systems.md]
│   ├── Math Prerequisites (linear algebra, calculus, probability, optimization)
│   ├── Classical ML (regression, trees, boosting, SVMs, unsupervised)
│   ├── Deep Learning (backprop, CNNs, LSTMs, Transformers, diffusion)
│   ├── LLMs (KV cache, attention complexity, tokenization, scaling laws)
│   ├── LLM Training (pre-training, SFT, RLHF, DPO, LoRA, quantization)
│   ├── LLM Inference (paged attention, continuous batching, speculative decoding)
│   ├── Distributed Training (DDP, FSDP, tensor/pipeline parallelism, ZeRO)
│   ├── MLOps (experiment tracking, model registry, serving, monitoring)
│   ├── Vector Databases & RAG (HNSW, chunking, hybrid search, re-ranking)
│   └── Recommendation Systems (CF, two-tower, sequential, retrieval+ranking)
│
├── L9: Security ───────────────────────────────── [10-security.md]
│   ├── AuthN & AuthZ (OAuth 2.0, OIDC, JWT, SAML, passkeys, Kerberos)
│   ├── Cryptography (AES, RSA, ECC, TLS internals, HSMs, KMS)
│   ├── Application Security (OWASP Top 10, SAST/DAST, supply chain)
│   ├── Infrastructure Security (WAF, DDoS, secrets management, RBAC)
│   └── Advanced (homomorphic encryption, MPC, ZK proofs, post-quantum)
│
└── L10: Cutting Edge ──────────────────────────── [11-cutting-edge.md]
    ├── Blockchain & Web3 (consensus, EVM, Layer 2, rollups, IPFS)
    ├── Federated Learning (FedAvg, differential privacy, on-device ML)
    ├── Quantum Computing (Shor's, Grover's, post-quantum cryptography)
    ├── AI Frontier (agents, multi-modal, world models, AI hardware)
    ├── Edge Computing (WASM, 5G MEC, Cloudflare Workers)
    └── Emerging Databases (HTAP, serverless DB, multi-model)
```

---

## Detailed Files

| # | File | Topics Covered |
|---|------|---------------|
| 01 | [cs-and-math-foundations.md](01-cs-and-math-foundations.md) | Math, DS, Algorithms, OS, Computer Architecture |
| 02 | [networking.md](02-networking.md) | OSI, TCP, HTTP, TLS, WebRTC, CDN, Load Balancing |
| 03 | [storage-and-databases.md](03-storage-and-databases.md) | RDBMS, NoSQL, LSM, Caching, Distributed Storage |
| 04 | [distributed-systems-theory.md](04-distributed-systems-theory.md) | CAP, Consensus, CRDTs, Fault Tolerance, Transactions |
| 05 | [infrastructure-and-devops.md](05-infrastructure-and-devops.md) | Docker, Kubernetes, Cloud, CI/CD, Observability |
| 06 | [architecture-patterns.md](06-architecture-patterns.md) | Microservices, Event-Driven, Kafka, Reliability |
| 07 | [specialized-systems.md](07-specialized-systems.md) | Search, Geo, Realtime, Media, Payments, Scheduling |
| 08 | [data-engineering.md](08-data-engineering.md) | Spark, Flink, Warehouses, Lakes, Feature Stores |
| 09 | [ai-ml-systems.md](09-ai-ml-systems.md) | Classical ML, Deep Learning, LLMs, MLOps, RAG |
| 10 | [security.md](10-security.md) | Auth, Crypto, AppSec, InfraSec, Post-Quantum |
| 11 | [cutting-edge.md](11-cutting-edge.md) | Blockchain, Federated Learning, Quantum, AI Frontier |

---

## Prerequisites Graph

```
L0 (Foundations)
    ↓
L1 (Networking) ──────────────────────────────┐
    ↓                                         │
L2 (Storage)                                  │
    ↓                                         ↓
L3 (Distributed Theory) ──────────────── L4 (Infrastructure)
    ↓                                         ↓
    └──────────────── L5 (Architecture Patterns) ─────────┐
                             ↓                            │
              ┌──────────────┼──────────────┐            │
              ↓              ↓              ↓            ↓
         L6 (Specialized) L7 (Data Eng)  L8 (AI/ML)  L9 (Security)
              └──────────────┴──────────────┴────────────┘
                                   ↓
                            L10 (Cutting Edge)
```

---

## Learning Time Estimates

| Level | Depth | Estimated Time |
|-------|-------|---------------|
| L0: CS & Math | Prerequisite | 4–8 weeks (or review if already known) |
| L1: Networking | Core | 2–3 weeks |
| L2: Storage | Core | 3–4 weeks |
| L3: Distributed Theory | Core | 3–4 weeks |
| L4: Infrastructure | Applied | 2–3 weeks |
| L5: Architecture Patterns | Applied | 3–4 weeks |
| L6: Specialized Systems | Applied | 4–6 weeks |
| L7: Data Engineering | Specialized | 3–4 weeks |
| L8: AI/ML Systems | Specialized | 8–12 weeks |
| L9: Security | Cross-cutting | 3–4 weeks |
| L10: Cutting Edge | Advanced | Ongoing |

---

## Key Resources to Pair With This Roadmap

- **Designing Data-Intensive Applications** — Martin Kleppmann (covers L2, L3, L5)
- **Computer Networks: A Top-Down Approach** — Kurose & Ross (covers L1)
- **The Algorithm Design Manual** — Skiena (covers L0)
- **Database Internals** — Alex Petrov (covers L2, L3)
- **Understanding Distributed Systems** — Roberto Vitillo (covers L3, L5)
- **Fundamentals of Data Engineering** — Reis & Housley (covers L7)
- **Deep Learning** — Goodfellow, Bengio, Courville (covers L8)
- **Building Machine Learning Powered Applications** — Ameisen (covers L8, MLOps)
