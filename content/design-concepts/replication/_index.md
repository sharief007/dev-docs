---
title: 'Database Replication'
weight: 1
type: docs
toc: true
sidebar:
  open: true
prev: 
next:
params:
  editURL:
---

{{< cards >}}
    {{< card link="read-replicas" title="Read Replicas & Replication Lag" subtitle="Sync vs async replication, replication lag, read-your-writes, monotonic reads, and replica promotion" >}}
    {{< card link="leader-based-replication" title="Leader-Based Replication" subtitle="Single-leader write path, follower catch-up, and failover" >}}
    {{< card link="multi-leader-replication" title="Multi-Leader Replication" subtitle="Write conflicts, conflict resolution, and cross-datacenter topologies" >}}
    {{< card link="replication-lag" title="Replication Lag" subtitle="Read-your-writes, monotonic reads, and consistent prefix reads" >}}
    {{< card link="consistency-models" title="Consistency Models" subtitle="Strong, eventual, causal, and linearizability" >}}
    {{< card link="replication-implementations" title="Replication Implementations" subtitle="Statement-based, WAL shipping, row-based, and logical replication" >}}
{{< /cards >}}


