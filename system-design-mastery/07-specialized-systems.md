# L6: Specialized Systems

> Real-world system design problems require domain-specific knowledge. These patterns appear in FAANG-level design interviews and real production systems.

---

## 1. Search Systems

### Inverted Index Construction

```
Documents:
  Doc1: "the quick brown fox"
  Doc2: "the lazy brown dog"

Inverted Index:
  "the"   → [Doc1:0, Doc2:0]
  "quick" → [Doc1:1]
  "brown" → [Doc1:2, Doc2:2]
  "fox"   → [Doc1:3]
  "lazy"  → [Doc2:1]
  "dog"   → [Doc2:3]
```

**Tokenization**: split text into tokens (whitespace, punctuation, special chars)
**Normalization**: lowercase, remove accents
**Stemming**: reduce words to root form (running → run, dogs → dog)
**Lemmatization**: linguistically correct root (ran → run, better → good)
**Stop words**: remove common words (the, a, is) — reduces index size

### Relevance Ranking

**TF-IDF** (Term Frequency — Inverse Document Frequency):
- TF(t, d) = count of term t in document d / total terms in d
- IDF(t) = log(N / df(t)) where N=total docs, df=docs containing t
- TF-IDF(t, d) = TF × IDF — high for rare terms that appear often in doc

**BM25** (Best Match 25) — Elasticsearch default:
- Improves TF-IDF with document length normalization and saturation
- score = Σ IDF(t) × (TF(t,d) × (k1+1)) / (TF(t,d) + k1×(1 - b + b×(dl/avgdl)))
- k1 (1.2–2.0): TF saturation, b (0.75): length normalization

### Lucene Segments (Elasticsearch Internals)
- Each index = collection of immutable **segments**
- Each segment is a complete, searchable mini-index
- New documents → new segment (memory buffer first)
- Background **merge**: multiple small segments merged into larger ones
- Merge reduces segment count → faster searches
- Deletion: mark as deleted in `.del` file, physically removed during merge

### String Algorithms for Search

**KMP (Knuth-Morris-Pratt)**: O(n+m) single pattern
- Precompute failure function: longest proper prefix that's also a suffix
- Never back up in text

**Rabin-Karp**: rolling hash for pattern matching
- Hash window of size m, roll forward
- Good for multiple patterns with same length

**Aho-Corasick**: O(n + m_total + k) multi-pattern matching
- Build trie of all patterns
- Add failure links (like KMP failure function across trie)
- Single pass over text matches all patterns simultaneously
- Used in: antivirus, network intrusion detection, ad keyword matching

**Suffix Array**: lexicographically sorted array of all suffixes
- Build: O(n log n) with doubling trick, O(n) with SA-IS algorithm
- LCP array: longest common prefix between adjacent suffixes
- Space: O(n) — much better than suffix tree

**Suffix Tree + Ukkonen's Algorithm**:
- Compacted trie of all suffixes: O(n) space
- Ukkonen's builds it online (left-to-right) in O(n) time
- Supports: substring search O(m), longest repeated substring, longest common substring
- More powerful but harder to implement than suffix array

**Autocomplete/Type-ahead**:
- Trie with popularity scores
- Top-k queries per prefix
- Compressed trie (Patricia tree) for memory efficiency
- Fuzzy matching: BK-tree or Levenshtein automaton for spell tolerance

### Near-Duplicate Detection

**MinHash + LSH** (Locality-Sensitive Hashing):
- MinHash: random permutations → probability of collision = Jaccard similarity
- LSH: group similar documents into same bucket (band technique)
- O(n) approximate duplicate detection
- Used in: web crawlers (dedup pages), ad duplicate detection, plagiarism detection

**SimHash** (Google):
- Fingerprint that similar documents have similar fingerprints
- Hamming distance between fingerprints ∝ similarity
- Used in: Google web crawl deduplication

### Semantic Search (Dense Retrieval)

- Encode query and documents into dense vectors (embeddings)
- Cosine similarity or dot product to find nearest neighbors
- Requires: embedding model (BERT, sentence-transformers) + vector database (HNSW)
- **Hybrid search**: combine BM25 (lexical) + dense (semantic) scores (Reciprocal Rank Fusion)

---

## 2. Location & Geospatial Systems

### Coordinate Systems
- **WGS84**: standard GPS coordinate system (latitude, longitude)
- Latitude precision: 1° ≈ 111km, 0.0001° ≈ 11m, 0.000001° ≈ 11cm
- Longitude precision varies with latitude (cos(lat) factor)

### GeoHashing

**Algorithm**:
1. Interleave bits of latitude and longitude binary representations
2. Encode in base32 (0-9, b-z minus a,i,l,o)
3. Longer hash = higher precision

**Precision levels**:
```
Length | Width  | Height
1      | 5000km | 5000km
3      | 156km  | 156km
5      | 4.9km  | 4.9km
7      | 153m   | 153m
9      | 4.8m   | 4.8m
```

**Neighbor cells**: each cell has 8 neighbors with known prefix patterns
**Proximity search**: get N-character geohash, search that cell + 8 neighbors
**Problem**: edge effects at cell boundaries (two points close in space but different geohash)
**Solution**: always search 9 cells (center + 8 neighbors)

### QuadTree

- Recursively divide 2D space into 4 quadrants
- Each leaf node covers a region with ≤ N points
- Used for: spatial indexing, map tiles, dynamic spatial decomposition
- Good for: non-uniform point distribution (unlike fixed geohash grid)

### R-Tree

- Bounding rectangle tree for arbitrary 2D shapes
- Minimum Bounding Rectangles (MBR) group nearby objects
- Used in: PostGIS, SQLite SpatiaLite, RDBMS spatial indexes
- Supports: point, polygon, line, rectangle queries

### Uber H3 (Hexagonal Hierarchical Spatial Index)

- Earth divided into hexagons at 15 resolution levels
- Hexagons are more uniform than squares (equal distance to all 6 neighbors)
- O(1) containment check, neighbor lookup
- **Use case**: Uber surge pricing regions, driver density visualization
- Open source: h3geo.org

### Google S2 Geometry

- Maps Earth surface to 1D using Hilbert curve (space-filling curve)
- Cells at multiple resolutions (S2Cells)
- **Hilbert curve**: nearby cells on 1D curve are nearby in 2D space → excellent range scans
- Used in: Google Maps, Foursquare, many geospatial databases

### KD-Tree

- Binary tree for k-dimensional data
- Each level splits on one dimension (cycling through dimensions)
- Nearest neighbor search: O(log n) average
- **Problem**: degrades to O(n) in high dimensions ("curse of dimensionality")
- Good for: 2D/3D spatial queries, robotics

### Route Optimization

**Dijkstra's Algorithm**: optimal but slow for large road networks (millions of nodes)

**A* Search**: Dijkstra + heuristic (straight-line distance to goal)
- Faster but requires admissible heuristic
- Good for small road networks

**Contraction Hierarchies (CH)**:
- Preprocessing: rank nodes, create "shortcuts" between high-rank nodes
- Query: bidirectional Dijkstra on contracted graph
- 10,000× faster than plain Dijkstra for road networks
- Used in: OSRM, Google Maps, Bing Maps

**Real-Time Location Tracking at Scale (Uber)**:
- Driver → writes location to Cassandra/Redis every 4 seconds
- Geohash index for finding nearby drivers
- Driver-supply service: maintains in-memory geohash → driver mapping
- Rider app: queries nearby drivers via geohash lookup

### Geofencing

- Trigger event when entity enters/exits geographic area
- Implementation: point-in-polygon algorithm
  - Ray casting: draw ray from point, count intersections with polygon
  - Winding number: more robust for complex polygons
- At scale: store geofences indexed by geohash, check only relevant geofences

---

## 3. Real-Time Communication

### WebSocket Architecture at Scale

**Problem**: WebSocket connections are stateful (long-lived TCP connections)
- Can't just put WebSocket servers behind a dumb load balancer
- Need sticky sessions (same client → same WebSocket server)

**Pattern**:
```
Client ──→ Load Balancer (sticky sessions) ──→ WebSocket Server
                                                      ↕ (Redis Pub/Sub)
                                              Other WebSocket Servers
```

- When server A needs to send message to user on server B: publish to Redis → all servers consume
- **Redis Pub/Sub** or **NATS** for cross-node message passing

**Connection Management**:
- Heartbeat/ping-pong to detect stale connections
- Reconnection logic in client with exponential backoff
- Connection token for authentication (not cookies — cookies don't work for WebSocket handshake)

### MQTT Protocol

- ISO standard publish-subscribe protocol for IoT
- Very lightweight (2-byte header)
- QoS levels: 0 (at most once), 1 (at least once), 2 (exactly once)
- **Retain**: last message stored, delivered to new subscribers
- **Will message**: auto-published on unexpected disconnect
- Used by: IoT devices, MQTT brokers (Mosquitto, EMQX, AWS IoT Core)

### Presence Systems (Online/Offline)

**Design**:
- User connects → write to Redis `SET user:{id}:online 1 EX 60`
- Heartbeat every 30s to refresh TTL
- No heartbeat → TTL expires → user is offline
- Fan-out: when user goes online/offline, notify their friends

**At scale** (billions of users):
- Shard users across many Redis instances
- Use consistent hashing to route to correct shard
- Batch presence updates to avoid write storms

### Message Ordering and Deduplication

**Total ordering**: use Kafka partition — all messages for same user in same partition
**Deduplication**: message ID + idempotency check before processing
**Sequence numbers**: client assigns sequence number, detect gaps, request retransmission

### Chat System Design (WhatsApp/Slack Architecture)

**Key Design Decisions**:
- Message storage: Cassandra (key=channel_id+timestamp, value=message)
- Fan-out: 1-1 chat (2 recipients), group chat (N recipients)
- **Fan-out on write**: write to each recipient's inbox at send time — fast reads, slow writes for large groups
- **Fan-out on read**: store once, each recipient's client reads and applies since last_seen — fast writes, slow reads
- Hybrid: fan-out on write for small groups (<100), fan-out on read for celebrities/large groups

**Message States**: sent → delivered → read (WhatsApp double blue checkmarks)
- Track per-user delivery receipts in DB

---

## 4. Notification Systems

### Push Notification Infrastructure

- **APNs** (Apple Push Notification Service): for iOS
- **FCM** (Firebase Cloud Messaging): for Android and web
- **Provider**: your server → APNs/FCM → device

**Fan-out Problem**:
- Twitter celebrity (100M followers) posts → 100M notifications
- **Fan-out on write**: compute fan-out at write time (precompute notification lists)
- **Fan-out on read**: each follower fetches new posts on open (celebrities only)
- **Hybrid**: small accounts do write fan-out, celebrities do read fan-out

**Notification Templates**: centralized service renders message + subject + channel-specific format

### Email at Scale

**SPF**: which IPs can send email for your domain (prevents spoofing)
**DKIM**: cryptographic signature on email headers (proves email unmodified)
**DMARC**: policy for what to do if SPF/DKIM fail (reject, quarantine, none)
**Deliverability**: warm up IPs gradually, monitor bounce rates, remove invalid addresses

---

## 5. Payment Systems

### Idempotency in Payments

- Every payment API call gets an idempotency key (UUID from client)
- Server stores: `{idempotency_key → response}` for 24 hours
- Retry same key → return same response (no double charge)
- Critical: prevents double-charging on network timeout

### Double-Spend Prevention

- Use database transactions with `SELECT FOR UPDATE` (pessimistic locking)
- Or optimistic locking with version numbers
- Ledger entries: never update, only append (debit/credit records)

### Ledger System (Double-Entry Bookkeeping)

```
Every transaction: one debit entry + one credit entry
Balance = sum(credits) - sum(debits) for an account

Example: Alice pays $100 to Bob
  DEBIT  Alice  -$100
  CREDIT Bob    +$100
```

- Immutable append-only ledger
- Balance computed by aggregation (or cached + maintained)
- Audit: every dollar can be traced

### Reconciliation

- Compare internal ledger with external payment processor records
- Run daily/hourly batch job
- Exceptions: missing charges, duplicate charges, amount mismatches
- Alert on reconciliation failures

---

## 6. Scheduling & Workflow Systems

### Job Queue Architecture

**Basic Queue**: Redis List, SQS, RabbitMQ
- Worker polls for jobs, processes, acks
- Dead letter queue for failed jobs after N retries

**Job Deduplication**:
- Hash job parameters → check if in-flight or recently completed
- Redis SET `NX PX 3600000` on job hash

**Priority Queues**:
- Redis Sorted Sets (ZADD with priority score)
- Multiple queues with different polling weights

**Delay Queues**:
- Schedule job for future time
- Redis: ZADD with timestamp as score, poll with ZRANGEBYSCORE

### Distributed Cron

**Problem**: if you run cron on multiple servers, jobs run multiple times
**Solutions**:
- **Single cron server**: SPOF, simple
- **Distributed lock**: acquire lock before running job (Redis/ZooKeeper)
- **Quartz Scheduler** (Java): cluster-aware, stores job state in DB
- **Kubernetes CronJob**: built-in deduplication, restartPolicy

### Workflow Engines

**Apache Airflow**:
- DAGs (Directed Acyclic Graphs) define workflows in Python
- Scheduler: triggers tasks when dependencies satisfied
- Workers: execute tasks (Celery, KubernetesExecutor)
- Rich UI with DAG visualization, task history
- Use cases: ETL pipelines, ML training pipelines, data quality checks

**Temporal**:
- Durable workflow execution (persists state to DB on every step)
- Workflows survive crashes, server restarts, deploys
- Coded in regular programming languages (Go, Java, Python, TypeScript)
- Activities (external calls) are automatically retried
- Better than Airflow for: complex logic, long-running workflows (months), microservice orchestration

**Prefect / Dagster**: modern alternatives to Airflow with better testing and data-aware scheduling

### Fair Scheduling (Multi-tenancy)

**FIFO**: simple but unfair (large jobs block small ones)
**Round Robin**: rotate between queues
**Fair Queue**: each user gets equal share of resources
**Weighted Fair Queue**: users get shares proportional to their weight
**CFS** (Completely Fair Scheduler — Linux): virtual runtime tracking
**DRF** (Dominant Resource Fairness — Mesos, YARN): multi-resource fairness (CPU + memory)
