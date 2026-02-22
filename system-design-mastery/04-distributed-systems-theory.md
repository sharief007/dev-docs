# L3: Distributed Systems Theory

> The theoretical foundations that explain *why* distributed systems behave the way they do. Without this, you're cargo-culting patterns without understanding their limits.

---

## 1. Fundamental Limitations

### The 8 Fallacies of Distributed Computing (Peter Deutsch, Sun Microsystems)
1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

Every one of these is false in production. Design accordingly.

### The Two Generals Problem
- Two armies must coordinate an attack, communicating only via messengers who may be captured
- **Proof**: No protocol over unreliable channels can guarantee agreement
- **Implication**: In distributed systems over unreliable networks, you can never be 100% certain the other party received your message
- TCP's three-way handshake doesn't solve this — it just makes failure probability very low

### The Byzantine Generals Problem (Lamport, Shostak, Pease — 1982)
- Generals must agree on attack/retreat, some generals are traitors (send conflicting messages)
- **Byzantine failure**: a node acts arbitrarily — sends different messages to different peers, lies
- **Byzantine fault tolerance (BFT)**: a system that functions correctly even with up to f traitors requires n ≥ 3f + 1 nodes
- **Crash fault tolerance** (CFT): simpler — assumes failed nodes just stop (don't lie)
  - Paxos, Raft handle crash faults (f failures tolerated with 2f+1 nodes)
  - PBFT, HotStuff handle Byzantine faults (requires 3f+1 nodes)

### CAP Theorem (Brewer, 2000)
**In the presence of a network Partition, you must choose between Consistency and Availability**

- **Consistency (C)**: every read receives the most recent write or an error
- **Availability (A)**: every request receives a non-error response (but may be stale)
- **Partition Tolerance (P)**: system continues operating despite network partition

**Practical implications**:
- Partitions *will* happen — you cannot avoid P
- Therefore: you choose between CP (sacrifice availability) or AP (sacrifice consistency)
- **CP systems**: HBase, ZooKeeper, etcd, Spanner — return error when partitioned
- **AP systems**: DynamoDB, Cassandra, CouchDB — return stale data when partitioned

**CAP is often misunderstood**:
- It's about partition tolerance, not "pick any 2"
- C in CAP = linearizability (strong), not just "consistency"
- When no partition: all systems can provide both C and A

### PACELC Model (Abadi, 2012)
Extends CAP to address the latency tradeoff even when no partition exists:
```
If Partition:  choose between Availability vs Consistency
Else (no partition): choose between Latency vs Consistency
```

- **PA/EL**: DynamoDB, Cassandra — always prefer availability and low latency
- **PC/EC**: Spanner, HBase — always prefer consistency (higher latency)
- **PA/EC**: MongoDB (primary prefers consistency, but secondaries may lag)

### FLP Impossibility (Fischer, Lynch, Paterson — 1985)
**In an asynchronous system, no deterministic consensus algorithm can tolerate even one crash failure.**

- Proof: in asynchronous model, you can't distinguish a crashed node from a slow one
- **Implication**: real consensus protocols (Paxos, Raft) get around this by using timeouts/randomization or making synchrony assumptions
- Why it matters: explains why Raft uses election timeouts, why ZooKeeper needs heartbeats

### BASE Properties (vs ACID)
- **Basically Available**: system remains available even during partial failures
- **Soft state**: data may change without input (replication updates)
- **Eventually consistent**: given no new updates, all replicas converge

---

## 2. Consistency Models

From **strongest** (most expensive) to **weakest** (highest performance):

### Linearizability (Strict Consistency)
- Operations appear to execute instantaneously at some point between invoke and return
- All operations form a total order consistent with real time
- **The gold standard** — what most people mean when they say "consistent"
- Used by: etcd, ZooKeeper, Google Spanner, single-leader Redis with sync replication
- Implemented via: consensus protocols (Paxos, Raft)

### Sequential Consistency (Lamport, 1979)
- All operations appear to execute in *some* sequential order
- Each process's operations appear in program order
- Weaker than linearizability: operations don't need to appear in real-time order
- Not composable (unlike linearizability)

### Causal Consistency
- Causally related operations must be seen in order by all nodes
- Concurrent (causally unrelated) operations can be seen in any order
- Implemented via: vector clocks, causal metadata
- Used in: MongoDB (causally consistent sessions), COPS system

### PRAM Consistency (Pipeline RAM)
- Each process sees its own writes in order
- Different processes may see writes in different orders
- Weaker than causal — doesn't track cross-process causality

### Session Guarantees (Terry et al.)
- **Read Your Writes**: once you write, you always see that write (even on other replicas)
- **Monotonic Reads**: if you've seen a value, you never see an older value
- **Writes Follow Reads**: if you read X then write Y, Y is ordered after X
- **Monotonic Writes**: your writes are applied in order
- Implemented by: sticky sessions, or tracking a "watermark" per client

### Eventual Consistency
- If no new updates, all replicas will eventually converge
- No guarantees about which value you read in the interim
- Performance: reads never stall, writes always succeed
- Used by: DNS, Cassandra (at low consistency levels), DynamoDB (eventually consistent reads)

---

## 3. Time & Ordering

### The Problem with Physical Clocks
- No two clocks run at exactly the same rate (clock drift ~1ms/min)
- NTP synchronizes to ~1–50ms accuracy (bad for distributed ordering!)
- GPS-disciplined clocks get to ~1μs but are expensive

### Lamport Timestamps (1978)
```
Rules:
1. Each event increments own clock: C = C + 1
2. On send: attach C to message
3. On receive: C = max(C_self, C_received) + 1
```
- Gives partial order: if A → B (A happened before B), then L(A) < L(B)
- **But**: L(A) < L(B) does NOT mean A happened before B
- Cannot determine causal relationship from timestamps alone

### Vector Clocks (Fidge/Mattern, 1988)
```
Each process i maintains V[1..n]
Rules:
1. On local event: V[i]++
2. On send: attach V
3. On receive: V[j] = max(V[j], received[j]) for all j; V[i]++
```
- Can determine causality: V(A) ≤ V(B) iff A causally precedes B
- **Problem**: size grows with number of processes → dotted version vectors optimize this
- Used in: Riak, DynamoDB conflict detection, debugging tools

### Version Vectors
- Like vector clocks but track updates to objects, not events
- Used for detecting conflicts in multi-master replication
- DynamoDB-style: {(server_id, counter)...} per object

### Hybrid Logical Clocks (HLC)
- Combines physical time with logical clock
- HLC = max(physical time, HLC + 1)
- Stays close to physical time while providing total ordering
- Used in: CockroachDB, YugabyteDB

### TrueTime (Google Spanner)
- Uses atomic clocks + GPS at each datacenter
- Returns interval [earliest, latest] bounding true time
- **Commit wait**: before committing, wait until `now.earliest > commit_timestamp`
  - Ensures causality: a transaction that starts after another transaction commits will have a later timestamp
- Typical uncertainty: 1–7ms

### Total Order Broadcast
- All messages delivered to all nodes in the same total order
- Equivalent to consensus: if you can solve one, you can solve the other
- Used as the primitive for: replicated state machines, distributed logs (Kafka)

---

## 4. Consensus Algorithms

### Paxos (Lamport, 1989)

**Roles**: Proposer, Acceptor, Learner

**Phase 1 (Prepare)**:
1. Proposer sends `PREPARE(n)` to majority of acceptors
2. Acceptor: if n > max seen, promises not to accept anything < n, returns highest accepted value

**Phase 2 (Accept)**:
1. Proposer sends `ACCEPT(n, value)` — value is from highest-numbered accepted proposal, or proposer's own if none
2. Acceptor: if no newer promise, accepts and sends back ACCEPTED(n, value)

**Commit**: once majority accept, value is chosen (learned)

**Multi-Paxos**: elect stable leader, skip Phase 1 for subsequent rounds
**Limitations**: hard to understand, no reference implementation in paper

### Raft (Ongaro & Ousterhout, 2014)

Designed for understandability. Same safety guarantees as Multi-Paxos.

**Leader Election**:
- All nodes start as Followers
- Follower → Candidate if no heartbeat within election timeout (randomized 150–300ms)
- Candidate requests votes from peers
- Wins if majority vote, becomes Leader
- Term numbers prevent split votes from persisting

**Log Replication**:
- Leader receives client requests, appends to its log
- Sends AppendEntries RPC to followers
- Once majority acknowledge, commit (advance commit index)
- Apply to state machine

**Safety Guarantee**: a leader always has all committed entries (RequestVote only votes for candidates with logs at least as up-to-date)

**Log Compaction (Snapshots)**:
- Log grows unbounded → take snapshot of state machine at some point
- Truncate log before snapshot point
- Use InstallSnapshot RPC to catch up lagging followers

**Membership Changes**:
- Joint consensus: transition through a configuration that requires majority of both old and new config
- Single-server changes: safer, only add/remove one server at a time

### ZAB (ZooKeeper Atomic Broadcast)
- Used exclusively in ZooKeeper
- Similar to Raft, but optimized for ZooKeeper's primary-backup model
- Uses epoch numbers (like terms) and ZXIDs (transaction IDs)
- **Recovery mode vs broadcast mode**
- ZAB guarantees: sequential consistency (not linearizability by default without `sync`)

### PBFT (Practical Byzantine Fault Tolerance — Castro & Liskov, 1999)
- Handles Byzantine failures (malicious/faulty nodes)
- Requires 3f+1 nodes to tolerate f Byzantine failures
- **Three phases**: pre-prepare, prepare, commit (quadratic message complexity O(n²))
- Used in: Hyperledger Fabric, some early blockchain systems
- **Limitation**: doesn't scale beyond ~20 nodes due to message complexity

### HotStuff (Used in Diem/Libra, Tendermint-inspired)
- Byzantine fault tolerant with linear message complexity O(n)
- Uses threshold signatures to aggregate votes
- Pipelined commit: can commit in 2 rounds per block
- Used in: Facebook Diem, many modern BFT blockchains

### EPaxos (Egalitarian Paxos — multi-leader Paxos)
- No single leader — any node can propose
- Commands on different keys can be committed in parallel
- Commands on same key go through traditional Paxos ordering
- Better throughput for geo-distributed systems

---

## 5. CRDTs — Conflict-free Replicated Data Types

> Data structures that can be updated concurrently and merged without conflicts.

### Why CRDTs
- In distributed systems with network partitions, you need to accept writes on multiple nodes
- When nodes reconnect, you need to merge state without losing data
- CRDTs provide mathematical guarantee of convergence

### Types

**State-based (CvRDT — convergent)**
- Nodes gossip full state, merge using join (least upper bound)
- Merge is idempotent, commutative, associative

**Operation-based (CmRDT — commutative)**
- Nodes broadcast operations, which must be delivered reliably and applied once
- Operations must be commutative (order doesn't matter)

### Common CRDTs

**G-Counter (Grow-only Counter)**
- Each node has its own counter slot: V[nodeID]++
- Merge: max(V1[i], V2[i]) for all i
- Read: sum(V[i])

**PN-Counter (Positive-Negative Counter)**
- Two G-Counters: one for increments, one for decrements
- Read: P.sum - N.sum

**G-Set (Grow-only Set)**
- Only additions, no removals
- Merge: union

**2P-Set (Two-Phase Set)**
- Add-set + Remove-set; removed elements can never be re-added
- Query: element in Add-set but not Remove-set

**OR-Set (Observed-Remove Set)**
- Tag each element with unique ID on add
- Remove removes specific tagged instances
- Re-adding works (new unique tag)
- Used in: Riak, shopping carts

**LWW-Element-Set (Last-Write-Wins)**
- Each element has a timestamp; latest write wins for add/remove
- **Risk**: timestamp ties → arbitrary resolution; requires good clock sync

**MV-Register (Multi-Value Register)**
- Stores multiple concurrent values (like a set)
- Client chooses which to keep (Git merge-like)
- Used in: DynamoDB, Riak (returns siblings on conflict)

**RGA (Replicated Growable Array — sequence CRDT)**
- Sequence CRDT for ordered text
- Each character assigned unique ID
- Insertions reference predecessor character by ID
- **Used in**: Yjs, Automerge for collaborative text editing (Google Docs-like)

**Yjs**
- Production-grade CRDT framework (JavaScript)
- Shared types: Y.Map, Y.Array, Y.Text, Y.XmlElement
- Encoding: optimized binary delta encoding

**Automerge**
- Haskell/JavaScript, JSON-like CRDT
- Good for complex document collaboration

---

## 6. Fault Tolerance Patterns

### Replication
- **Synchronous replication**: write confirmed only after replica acks — strong consistency, higher latency
- **Asynchronous replication**: write confirmed immediately, replica updated later — lower latency, potential data loss on failover
- **Semi-synchronous** (MySQL): at least one replica must ack before commit

### Quorum Systems
- N replicas, write to W, read from R
- **Strong consistency**: W + R > N (read set overlaps write set)
- Common: N=3, W=2, R=2 (quorum quorum)
- **Consistency levels** in Cassandra: ONE, QUORUM, LOCAL_QUORUM, EACH_QUORUM, ALL
- **Quorum with leaderless**: Dynamo-style (DynamoDB, Cassandra, Riak)

### Failure Detection
- **Heartbeat**: nodes send periodic pings, declare dead on timeout
- **Timeout tuning**: too short → false positives, too long → slow recovery
- **Phi Accrual Failure Detector** (Akka, Cassandra):
  - Adaptive threshold based on historical heartbeat intervals
  - Outputs continuous suspicion level (phi) not binary dead/alive
  - `phi = -log10(1 - CDF(heartbeat_interval))`
- **SWIM protocol** (Scalable Weakly-consistent Infection-style Membership):
  - Indirect ping: if direct ping fails, ask N random nodes to ping target
  - Reduces false positives, used in Consul, Serf, HashiCorp ecosystem

### Gossip Protocols
- Nodes randomly select peers and exchange information
- Information spreads epidemic-ally: O(log N) rounds to reach all nodes
- **Convergence**: within a few rounds
- Applications: membership, failure detection, data dissemination
- Used in: Cassandra (ring membership), Bitcoin (transaction propagation)

### Split-Brain Prevention
- **Fencing tokens**: monotonically increasing tokens, storage rejects requests with old token
- **STONITH** (Shoot The Other Node In The Head): forcibly terminate suspected node
  - Implemented via: IPMI, iLO, remote power management
  - Ensures only one node writes at a time (prevents split-brain in cluster)
- **Disk-based fencing**: acquire exclusive lock on shared storage

---

## 7. Distributed Coordination

### ZooKeeper
- Ordered log of changes (ZAB protocol)
- **Znodes**: hierarchical namespace like filesystem
  - Persistent, ephemeral (deleted when session ends), sequential
- **Watches**: one-time notifications on change
- **Use cases**:
  - Leader election (ephemeral znodes, sequence nodes)
  - Distributed locks (ephemeral sequential nodes)
  - Configuration management
  - Service registry

### etcd
- Raft-based, consistent key-value store
- Kubernetes control plane store (all cluster state)
- gRPC API (vs ZooKeeper's custom protocol)
- Watch API with revision numbers
- Leases: keys expire unless renewed (like ephemeral znodes)
- Transactions: multi-op with compare-and-swap

### Distributed Locks

**Simple Redis lock (Setnx)**
- `SET lock_key unique_value NX PX 30000`
- Release: `if GET lock_key == unique_value then DEL lock_key` (Lua atomic)
- Problem: if lock holder dies before expiry, lock stuck for TTL

**Redlock Algorithm (Antirez)**
- Lock on majority (3/5) Redis masters
- Contention: release all locks if can't acquire majority
- Validity time = min_validity - elapsed_time - clock_drift
- **Controversial**: Martin Kleppmann argues Redlock is unsafe for correctness-critical locks
  - Use fencing tokens instead when you need true mutual exclusion

**Chubby (Google)** — inspiration for ZooKeeper
- Coarse-grained distributed locks
- Advisory locks with sequencer (fencing token equivalent)

---

## 8. Distributed Transactions

### Two-Phase Commit (2PC)

```
Coordinator → Participants: PREPARE
Participants → Coordinator: VOTE_YES / VOTE_NO
  (if all YES) Coordinator → Participants: COMMIT
  (if any NO)  Coordinator → Participants: ROLLBACK
```

**Problems**:
- **Blocking**: if coordinator crashes between PREPARE and COMMIT, participants block forever holding locks
- **Failure during Phase 2**: uncertain transaction state

### Three-Phase Commit (3PC)
- Adds **PRE-COMMIT** phase between PREPARE and COMMIT
- Participants can now infer intent and recover without coordinator
- **Problem**: still fails under network partition (can't distinguish coordinator failure from partition)

### Saga Pattern

Long-running distributed transactions split into local transactions:

```
T1 → T2 → T3 → ... → Tn
           ↑ if Tk fails:
Cn → ... → Ck-1 (compensating transactions)
```

**Choreography**:
- Each service reacts to events from previous step
- No central coordinator
- Hard to track saga state

**Orchestration**:
- Central saga orchestrator sends commands
- Easier to monitor, but adds coupling

### Transactional Outbox Pattern

Problem: how to atomically write to DB AND publish to message queue?

```
In same DB transaction:
1. Write business data to main table
2. Write event to outbox table

Separate process:
3. Poll outbox table, publish to message queue
4. Mark events as published
```

Variations:
- **Polling publisher**: background job polls outbox
- **Transaction log tailing**: read WAL/CDC (Debezium) to capture outbox events — lower latency

### Change Data Capture (CDC)

- Capture database changes in real-time from transaction log
- **Debezium**: open-source CDC platform, connectors for PostgreSQL, MySQL, MongoDB, etc.
- Reads PostgreSQL WAL (logical decoding), MySQL binlog, MongoDB oplog
- Publishes changes to Kafka topics
- Use cases: event sourcing, cache invalidation, search index sync, data replication

### TCC (Try-Confirm-Cancel)
- Alternative to 2PC for distributed transactions
- **Try**: reserve/lock resources
- **Confirm**: commit the reservation
- **Cancel**: release the reservation
- Application-level implementation (no coordinator)
- Good for: booking systems (hotel, flights, car)
