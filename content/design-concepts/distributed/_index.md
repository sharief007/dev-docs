---
title: Distributed Systems
weight: 5
type: docs
toc: false
sidebar:
  open: true
---

{{< cards >}}
  {{< card link="cap-theorem" title="CAP Theorem" subtitle="Consistency, Availability, Partition Tolerance — the fundamental tradeoff in distributed systems" >}}
  {{< card link="pacelc" title="PACELC" subtitle="Extends CAP: during partitions choose A or C; else choose Latency or Consistency" >}}
  {{< card link="consistency-models" title="Consistency Models" subtitle="Linearizability, sequential, causal, eventual consistency, and session guarantees" >}}
  {{< card link="idempotency" title="Idempotency & Exactly-Once Semantics" subtitle="Safe retries, idempotency keys, delivery guarantees, Kafka exactly-once, deduplication patterns" >}}
  {{< card link="distributed-locking" title="Distributed Locking" subtitle="Redis SETNX, Redlock, fencing tokens, ZooKeeper znodes, and when not to use locks" >}}
  {{< card link="leader-election" title="Leader Election" subtitle="Bully algorithm, Raft election, ZooKeeper znodes, split-brain prevention, STONITH" >}}
  {{< card link="gossip-protocol" title="Gossip Protocol" subtitle="Fan-out convergence, phi accrual failure detection, SWIM, Merkle tree anti-entropy" >}}
  {{< card link="quorum" title="Quorum Reads & Writes" subtitle="R + W > N overlap rule, tuning configurations, sloppy quorum, hinted handoff, read repair" >}}
  {{< card link="saga-pattern" title="Saga Pattern" subtitle="Orchestration vs choreography, compensating transactions, execution log, and isolation countermeasures" >}}
  {{< card link="outbox-pattern" title="Outbox Pattern" subtitle="Dual-write problem, CDC with Debezium, at-least-once delivery, and inbox deduplication" >}}
  {{< card link="logical-clocks" title="Logical Clocks" subtitle="Lamport timestamps, vector clocks, version vectors, and Hybrid Logical Clocks" >}}
  {{< card link="multi-region-design" title="Multi-Region Design" subtitle="Active-passive, active-active, geo-routing, CRDTs, conflict resolution, and data sovereignty" >}}
{{< /cards >}}
