# L5: Architecture Patterns

> The patterns that appear repeatedly across all large-scale systems. Master these and you can design anything.

---

## 1. Estimation & Capacity Planning

### Latency Numbers to Memorize
```
L1 cache reference:           0.5 ns
L2 cache reference:           7 ns
Mutex lock/unlock:            25 ns
Main memory reference:        100 ns
Compress 1K with Snappy:      3 μs
Send 1K over 1Gbps network:   10 μs
Read 4K randomly from SSD:    150 μs
Read 1 MB sequentially SSD:   1 ms
Round trip in same datacenter: 0.5 ms
Read 1 MB sequentially HDD:   20 ms
HDD seek:                     10 ms
Packet CA→Netherlands→CA:     150 ms
```

### Back-of-Envelope Methodology
1. **Clarify scale**: DAU, QPS (peak vs average), storage growth rate
2. **Traffic estimation**: DAU × actions/day ÷ 86400 = avg QPS; multiply by 2–10× for peak
3. **Storage estimation**: entries/day × entry size × retention period
4. **Bandwidth estimation**: QPS × response size
5. **Cache estimation**: 20% of data serves 80% of traffic (Pareto principle)

### Common Reference Numbers
- 1 million users → ~11 QPS (assuming 1 request/day)
- Twitter: ~500M tweets/day → ~5,800/sec
- Instagram: ~100M photos/day → storage ~500TB/year
- Typical web server: handles ~5,000–50,000 RPS per core

---

## 2. API Design

### RESTful API Design

**Richardson Maturity Model**:
- Level 0: POX (Plain Old XML/JSON), single endpoint
- Level 1: Resources (multiple endpoints for resources)
- Level 2: HTTP verbs (GET/POST/PUT/DELETE correctly)
- Level 3: Hypermedia (HATEOAS — links to related actions)

**Best Practices**:
- Use nouns for resources: `/users/{id}/orders` not `/getOrdersForUser`
- Use HTTP methods correctly (idempotency matters)
- Return appropriate status codes (201 Created, 204 No Content, 409 Conflict, 422 Unprocessable)
- Versioning: URL path (`/v2/users`), header (`Accept: application/vnd.api+json;version=2`), query param
- Consistent error format: `{error: {code, message, field}}`

**Pagination Strategies**:
- **Offset**: `?page=2&size=20` — simple but slow for deep pages (DB scans offset rows)
- **Cursor**: `?after=eyJ...` — base64-encoded cursor (row ID or timestamp), O(1) regardless of depth
- **Keyset**: `?after_id=12345` — like cursor but readable, requires indexed field ordering

**Idempotency**:
- Idempotent: GET, HEAD, PUT, DELETE, OPTIONS — safe to retry
- Non-idempotent: POST — retrying may create duplicates
- **Idempotency-Key header**: client sends UUID, server deduplicates (used in Stripe, payment APIs)
- Store `(idempotency_key, response)` for ~24 hours

### GraphQL

- **Schema**: strongly typed, introspectable
- **Resolver**: function that fetches data for a field
- **N+1 Problem**: fetching 100 posts each triggers 100 user queries
  - **DataLoader**: batch and deduplicate database calls per request
- **Subscriptions**: real-time updates over WebSocket
- **Persisted Queries**: hash → query, reduces payload size
- **Tradeoffs vs REST**: flexible for complex UIs, harder to cache (POST requests), over-fetching solved

### Rate Limiting Algorithms

**Token Bucket**:
- Bucket holds N tokens, refills at rate R tokens/sec
- Each request consumes 1 token; reject if bucket empty
- Allows bursting up to bucket size
- **Best for**: API rate limiting with burst allowance

**Leaky Bucket**:
- Requests added to queue, processed at constant rate
- Queue full → drop request
- Smooths out bursts (no bursty traffic to backend)
- **Best for**: enforcing smooth output rate

**Fixed Window Counter**:
- Count requests in current time window (e.g., 1 minute)
- Reset at window boundary
- **Problem**: burst at window boundary (2× limit possible)

**Sliding Window Log**:
- Store timestamp of each request in sorted set
- Count requests in last N seconds
- Accurate but memory-intensive (stores all request times)

**Sliding Window Counter**:
- `count = current_window_count × (remaining_time/window) + prev_window_count`
- Approximation of sliding log with O(1) memory
- **Best for**: production rate limiting at scale (Redis-based)

### API Gateway

- Single entry point: authentication, authorization, rate limiting, logging, SSL termination
- **Kong**: open-source, Lua plugins
- **AWS API Gateway**: managed, Lambda integration, caching
- **Backend for Frontend (BFF)**: separate API layer per client type (mobile, web, third-party)
  - Mobile BFF optimized for bandwidth (fewer fields, offline handling)
  - Web BFF can assume fast network, richer responses

---

## 3. Microservices Architecture

### Domain-Driven Design (DDD)

**Strategic Design**:
- **Domain**: the problem space (e-commerce, banking, logistics)
- **Subdomain**: core, supporting, generic
  - Core domain: competitive advantage — invest here
  - Supporting subdomain: needed but not differentiating — build simply
  - Generic subdomain: commodity — buy/use off-the-shelf
- **Bounded Context**: explicit boundary where a particular domain model applies
  - "Product" means different things in catalog service vs warehouse vs billing

**Tactical Design** (inside a bounded context):
- **Entity**: has identity, lifecycle (User, Order, Product)
- **Value Object**: immutable, defined by values (Money, Address, Color)
- **Aggregate**: cluster of objects with a root entity; consistency boundary
  - All operations through aggregate root
  - Aggregates reference other aggregates by ID only (no direct object references across aggregates)
- **Domain Event**: something that happened (`OrderPlaced`, `PaymentFailed`)
- **Repository**: abstraction for aggregate persistence
- **Domain Service**: logic that doesn't belong to a specific entity
- **Anti-Corruption Layer (ACL)**: translate between bounded contexts

### Service Decomposition

**Decompose by subdomain**: one service per bounded context
**Decompose by business capability**: what the business does (order management, inventory, shipping)
**Decompose by volatility**: separate things that change at different rates

**Common Antipatterns**:
- **Distributed Monolith**: microservices that are tightly coupled (deploy together, share DB)
- **Chatty Services**: too many small services with high inter-service communication
- **Data coupling**: multiple services sharing the same database
- **Anemic domain model**: services with all logic in service layer, entities are just data

### Service Communication

**Synchronous (request-response)**:
- REST, gRPC
- Coupling: caller must wait for response
- Good for: queries, operations needing immediate confirmation

**Asynchronous (event-driven)**:
- Message queue / pub-sub
- Temporal decoupling: caller doesn't wait
- Good for: commands, workflows, fanout

**Service Discovery**:
- **Client-side**: service queries registry (Eureka), load balances itself (Ribbon)
- **Server-side**: load balancer queries registry (Kubernetes Services)
- **Self-registration** vs **3rd-party registration**

### Resilience Patterns

**Circuit Breaker** (Hystrix, Resilience4j):
- **Closed**: requests pass through normally
- **Open**: after N failures, stop sending requests (fail fast)
- **Half-open**: after timeout, allow one request to test if service recovered
- Benefits: fail fast (no cascade), give overloaded service time to recover

**Bulkhead Pattern**:
- Isolate different resources (thread pools, connection pools) per dependency
- Failure in one doesn't exhaust resources for others
- Named after ship compartments (one flooding doesn't sink the ship)

**Retry with Exponential Backoff + Jitter**:
```
wait = min(cap, base × 2^attempt) + random_jitter
```
- Without jitter: all retries synchronized → thundering herd
- Jitter: spread retries over time

**Timeout Pattern**:
- Every external call must have a timeout
- Cascading timeouts: upstream timeout < downstream sum (leave buffer for processing)

**Sidecar Pattern**:
- Deploy helper container alongside app container
- Handles: logging, tracing, proxy, config reloading
- App container doesn't change — concerns separated

---

## 4. Event-Driven Architecture

### Event Sourcing

**Core Idea**: store state as sequence of events, not current state snapshot
```
Events: UserCreated{id,email} → NameChanged{id, "Bob"} → EmailVerified{id}
State: rebuild by replaying events
```

**Benefits**:
- Complete audit log
- Point-in-time queries (replay to any moment)
- Easy event publishing (events are already there)
- Temporal queries (what did the state look like last Tuesday?)

**Challenges**:
- Querying current state requires replay (solved by projections)
- Schema evolution: events must be versioned
- Eventual consistency between write model and read projections

**Event Store**:
- Append-only log of domain events
- Event streams per aggregate
- Options: EventStoreDB, Kafka, PostgreSQL (with append-only constraint)

**Snapshotting**:
- Periodically save aggregate state to avoid replaying all events
- Start from latest snapshot, replay only newer events

### CQRS (Command Query Responsibility Segregation)

**Write side (Command)**:
- Validates and executes commands
- Updates event store (or write DB)
- Publishes domain events

**Read side (Query)**:
- Materialized views optimized for queries
- Updated by handling domain events (projections)
- Can have multiple read models optimized for different queries

**Eventual consistency**: read side may lag behind write side
**Read your writes**: solved by returning command ID, polling, or WebSocket notification

### Event Choreography vs Orchestration

**Choreography**: services react to events from other services
- Each service does its part and publishes events
- No central coordinator
- Pros: loose coupling, easy to extend
- Cons: hard to track overall saga state, hard to debug

**Orchestration** (Saga): central orchestrator sends commands to services
- Orchestrator knows the full workflow
- Easier to monitor and debug
- Cons: orchestrator knows about all services (more coupling)

### Inbox/Outbox Patterns

**Outbox Pattern** (transactional messaging):
```sql
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox (event_type, payload) VALUES ('OrderPlaced', '...');
COMMIT;
-- Background worker: read outbox, publish to message bus
```
- Atomic: order and event published together
- At-least-once delivery (deduplicate on consumer side)

**Inbox Pattern** (idempotent consumers):
```sql
BEGIN;
  INSERT INTO inbox (message_id) VALUES (?) ON CONFLICT DO NOTHING;
  -- if inserted (not duplicate): process message
COMMIT;
```
- Deduplication using message ID

**Dead Letter Queue (DLQ)**:
- Failed messages moved to DLQ after N retries
- Manual inspection and reprocessing
- Prevents poison pill messages from blocking queue

---

## 5. Messaging Systems

### Apache Kafka Deep Dive

**Core Concepts**:
- **Topic**: named log, messages are appended (never updated/deleted)
- **Partition**: horizontal scale unit, total ordering within partition only
- **Offset**: position of a message in a partition
- **Consumer Group**: multiple consumers reading from same topic, each partition assigned to one consumer
- **Replication**: leader partition + follower replicas per broker
- **In-sync replicas (ISR)**: replicas that are caught up with leader

**Producers**:
- `acks=0`: fire and forget (lowest latency, data loss possible)
- `acks=1`: leader acknowledges (leader failure → data loss)
- `acks=all / -1`: all ISR acknowledge (no data loss if min.insync.replicas ≥ 2)
- Batching: linger.ms + batch.size for throughput
- Compression: gzip, snappy, lz4, zstd

**Consumers**:
- Poll loop
- Auto-commit vs manual commit
- **At-most-once**: commit before processing
- **At-least-once**: commit after processing (most common)
- **Exactly-once**: Kafka transactions or idempotent consumer

**Log Compaction**:
- For changelog topics: keep only latest value per key
- Enables event sourcing-like "current state" topics
- Used for: CDC, materializing state

**Exactly-Once Semantics**:
- **Idempotent producer**: `enable.idempotence=true` — deduplicates retried messages
- **Transactions**: atomic writes across multiple partitions
  - `beginTransaction()` → produce → `commitTransaction()`
  - Consumers set `isolation.level=read_committed` to skip uncommitted messages

**Kafka Streams**:
- Java stream processing on top of Kafka
- Stateful operations (aggregations, joins) backed by RocksDB + changelog topics
- Processing guarantee: exactly-once (with Kafka transactions)

**Schema Registry** (Confluent):
- Central repository for Avro/Protobuf/JSON schemas
- Producers register schemas, consumers validate
- Schema evolution compatibility: BACKWARD, FORWARD, FULL

**Tiered Storage**:
- Offload old Kafka data to S3/GCS
- Brokers keep recent data, older data fetched from object storage on demand
- Reduces broker disk requirements dramatically

### RabbitMQ

**AMQP Protocol**:
- Producers → Exchange → Queue → Consumers

**Exchange Types**:
- `direct`: route by exact routing key
- `topic`: route by pattern (`orders.*.failed`)
- `fanout`: broadcast to all bound queues
- `headers`: route by message headers

**Message Acknowledgment**:
- Manual ack: consumer explicitly acks (or rejects)
- `nack` with requeue: put back in queue
- `nack` without requeue: goes to DLX (Dead Letter Exchange)

**Dead Letter Exchange (DLX)**: messages that fail, expire, or get rejected are routed here

**When to use RabbitMQ vs Kafka**:
- RabbitMQ: complex routing, short-lived messages, traditional task queues, push-based
- Kafka: durable log, high throughput, stream processing, event sourcing, long retention

### Apache Pulsar

- Unified messaging + streaming (replaces Kafka AND RabbitMQ use cases)
- **Separated storage** (BookKeeper) and **compute** (Brokers) — independently scalable
- **Multi-tenancy**: namespaces, topics, tenant isolation
- **Geo-replication**: built-in (Kafka requires MirrorMaker 2)
- **Functions**: lightweight compute (like Kafka Streams, simpler)
- Schema Registry built-in

### NATS.io

- Ultra-lightweight, low-latency messaging (~10μs)
- Publish-subscribe, request-reply, queue groups
- **NATS JetStream**: persistence, consumer acknowledgment (NATS + JetStream ≈ Kafka)
- Best for: low-latency microservice communication, IoT, edge

---

## 6. Serverless Architecture

### Function-as-a-Service (FaaS)

**AWS Lambda**:
- Event-driven: triggers (API Gateway, S3, DynamoDB Streams, Kinesis, SQS)
- Execution model: cold start + warm execution
- **Cold start**: download code, start runtime, initialize environment (~100ms–1s)
- **Warm start**: reuse existing execution environment (~1ms)
- Memory: 128MB–10GB (CPU proportional to memory)
- Timeout: max 15 minutes

**Cold Start Mitigation**:
- Provisioned concurrency (keep N warm)
- Minimize deployment package size
- Avoid VPC for latency-sensitive (ENI provisioning adds 10s to cold start)
- Snap Start (Lambda) / Lazy loading

**Serverless Limitations**:
- Stateless (no in-memory state between invocations)
- Concurrency limits (default 1000/account/region)
- Vendor lock-in

### AWS Step Functions

- State machine orchestrator for multi-step workflows
- Standard (60s timeout between steps) vs Express (high-volume, short duration)
- Handles: retries, error handling, parallel execution, wait states
- Use case: order processing pipeline, ML inference pipeline, ETL coordination

---

## 7. Reliability Patterns (Summary)

| Pattern | Problem Solved | Key Setting |
|---------|---------------|-------------|
| Circuit Breaker | Cascade failures | Failure threshold, reset timeout |
| Retry + Backoff | Transient failures | Max retries, base delay, jitter |
| Bulkhead | Resource exhaustion from one dependency | Thread pool per dependency |
| Timeout | Slow dependencies blocking threads | Per-hop timeout |
| Health Check | Route traffic away from unhealthy instances | `/health` endpoint |
| Graceful Degradation | Partial failure | Return cached/default on failure |
| Rate Limiting | Protect from overload | Token bucket, Redis counter |
| Idempotency | Duplicate requests | Idempotency-key in request |
| Saga | Distributed transaction consistency | Compensating transactions |
| Outbox | Atomic DB write + message publish | Outbox table + CDC |
