---
title: Consistency Models
weight: 2
type: docs
---

The consistency model your system provides is a direct consequence of how you configure replication. Choose synchronous replication and you get strong consistency — at the cost of availability and write latency. Choose asynchronous and you get low latency and high availability — but clients can observe stale or out-of-order data. Every production database sits somewhere on this spectrum, and the right answer depends on the operation.

## Synchronous Replication

The leader does not acknowledge a write until at least one (or all) replicas have confirmed they received and persisted the data. The client is guaranteed that any subsequent read from any replica returns the written value.

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2

    C->>L: COMMIT (write W)
    L->>L: fsync WAL to disk
    L->>F1: WAL record
    L->>F2: WAL record
    F1-->>L: ACK
    F2-->>L: ACK
    L->>C: COMMIT confirmed
    Note over C,F2: Strong consistency — all replicas current at commit time
```

**Guarantees:** zero replication lag, zero data loss on leader failure, linearizable reads from any replica.

**Cost:** write latency equals the round-trip time to the **slowest** replica. One unresponsive follower blocks all writes. This makes fully synchronous replication impractical for most production workloads.

## Semi-Synchronous Replication

A practical middle ground: the leader waits for **one** follower to ACK before confirming the write. The remaining followers replicate asynchronously.

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F1 as Follower 1 (sync)
    participant F2 as Follower 2 (async)

    C->>L: COMMIT (write W)
    L->>L: fsync WAL to disk
    L->>F1: WAL record
    L->>F2: WAL record
    F1-->>L: ACK
    L->>C: COMMIT confirmed
    Note over F2: ACK arrives later — may lag
```

If the synchronous follower fails, the leader promotes the next fastest async follower to become the new synchronous target. This guarantees that **at least two nodes** (leader + one follower) always hold every committed write.

**PostgreSQL configuration:**

```sql
-- Make one standby synchronous
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (standby1, standby2)';
-- Per-transaction override for non-critical writes
SET LOCAL synchronous_commit = 'local';  -- skip replica confirmation
```

**MySQL semi-sync** (MySQL 8.0.26+ uses the renamed `_source_` / `_replica_` system variables; the older `_master_` / `_slave_` names still work as deprecated aliases):

```sql
-- On the primary (MySQL 8.0.26+)
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_wait_for_replica_count = 1;
```

## Asynchronous Replication

The leader acknowledges the write as soon as it persists locally. Followers receive changes in the background and may lag behind by milliseconds, seconds, or — under load — minutes.

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2

    C->>L: COMMIT (write W)
    L->>L: fsync WAL to disk
    L->>C: COMMIT confirmed
    Note over L,F2: Background — followers may lag
    L-->>F1: WAL record
    L-->>F2: WAL record
    F1-->>L: ACK (later)
    F2-->>L: ACK (later)
```

**Guarantees:** lowest write latency, highest availability (one slow replica does not block writes).

**Risks:** clients reading from a lagging follower see stale data. If the leader fails before an unacknowledged write reaches any follower, that write is permanently lost.

This is the default mode in PostgreSQL (`synchronous_commit = on` only durably writes to the leader; standby replication is async unless configured otherwise) and MySQL (async replication is the default).

{{< callout type="warning" >}}
**"Eventually consistent" has no time bound.** Asynchronous replication guarantees that followers converge to the leader's state — but "eventually" could mean 50 ms or 5 minutes depending on replication throughput, network conditions, and follower load. Design read paths around this uncertainty: if something must be current, read from the leader or use [session guarantees](../read-replicas).
{{< /callout >}}

## Comparison

| Property | Fully Synchronous | Semi-Synchronous | Asynchronous |
|----------|------------------|------------------|-------------|
| Write latency | Slowest replica RTT | Fastest replica RTT | Local fsync only |
| Data loss on leader crash | None | None (1 replica guaranteed current) | Possible (unreplicated writes lost) |
| Read consistency (any replica) | Linearizable | Strong for sync follower; eventual for others | Eventual |
| Availability under replica failure | Blocked until recovery | Failover to next follower | Unaffected |
| Use case | Financial ledgers, metadata stores | Most OLTP databases | Analytics replicas, cross-region copies |

## Choosing the Right Mode

The correct answer is almost always **per-operation**, not per-system:

| Operation | Required Consistency | Replication Mode |
|-----------|---------------------|-----------------|
| Account balance before debit | Linearizable | Read from leader or sync replica |
| User profile after own edit | Read-your-writes | [Session guarantee](../read-replicas) or leader read |
| News feed timeline | Eventual | Read from nearest async replica |
| Inventory count before purchase | Strong | Synchronous or leader read with lock |
| Analytics dashboard | Eventual (seconds-stale acceptable) | Async read replica |

The theoretical underpinnings — linearizability, sequential consistency, causal consistency, eventual consistency — are covered in [Consistency Models (distributed)](../../distributed/consistency-models). This page focuses on how replication configuration delivers those guarantees in practice. For the anomalies that arise under asynchronous replication (stale reads, monotonic read violations, causal ordering violations), see [Read Replicas & Replication Lag](../read-replicas).

{{< callout type="info" >}}
**Interview tip:** When an interviewer asks about consistency in a replicated database, connect the replication mode to the consistency guarantee: "For the balance check, I'd read from the leader to get linearizable consistency — the write was synchronously replicated to one standby for durability, so failover is safe. For the activity feed, I'd read from the nearest async replica and tolerate seconds-stale data to keep read latency low." This shows you treat consistency as a per-operation cost decision, not a system-wide toggle.
{{< /callout >}}

## Test Your Understanding

{{< details title="A system uses eventual consistency. Two users update the same document simultaneously in different regions. Both succeed. Which version wins?" closed="true" >}}
**Conflict resolution determines the winner.** With eventual consistency, both writes are accepted (no coordination). When replicas converge, they must resolve the conflict. Common strategies:

- **Last-Write-Wins (LWW):** Highest timestamp wins. Simple but loses data — the other write is silently discarded. Clock skew can make a "later" write lose.
- **Application-level merge:** The application defines merge logic (e.g., union of set elements, concatenation of lists).
- **CRDTs (Conflict-free Replicated Data Types):** Data structures that mathematically guarantee convergence without coordination — counters, sets, registers.

The key insight: eventual consistency doesn't mean "data will be correct eventually" — it means "replicas will converge to the same value eventually." Whether that value is correct depends on the conflict resolution strategy.
{{< /details >}}

{{< details title="Causal consistency is weaker than linearizability but stronger than eventual. Give a concrete example where causal consistency is sufficient but eventual is not." closed="true" >}}
**Social media comments.** User A posts a comment. User B replies to A's comment. Under eventual consistency, some users might see B's reply before A's original comment — confusing and nonsensical.

Causal consistency guarantees that if B's reply **causally depends** on A's comment (B read A's comment before replying), then every node will see A's comment before B's reply. Concurrent, unrelated comments may appear in any order — that's fine.

Linearizability would be overkill here — it would require global coordination for every comment, adding latency. Causal consistency preserves the "makes sense" ordering without the performance cost of total ordering.
{{< /details >}}

{{< details title="A system claims 'strong consistency.' What question should you ask to determine if it's actually linearizable or just sequential?" closed="true" >}}
Ask: **"If I write a value at time T1, will a read starting at time T2 > T1 from any node always see that write?"**

- **Linearizable (strongest):** Yes. Operations appear to take effect at a single point in real time. A read that starts after a write completes must see it. Requires real-time ordering.
- **Sequential:** Operations appear in some total order consistent with each process's program order, but NOT necessarily in real-time order. Process A's write at T1 might not be visible to Process B's read at T2 if the sequential order places B's read before A's write.

Spanner is linearizable (TrueTime gives real-time ordering). Many "strongly consistent" systems are actually sequentially consistent — they guarantee a total order but not a real-time one.
{{< /details >}}