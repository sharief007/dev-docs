---
title: 'Multi Leader Replication'
weight: 4
type: docs
toc: true
sidebar:
  open: true
prev: 
next:
params:
  editURL:
---

Leader-based replication, with its single leader processing all writes, presents a significant drawback: if connectivity to the leader is lost due to network interruptions or other reasons, writing to the database becomes impossible.

An evolution of the leader-based model is the multi-leader configuration, also known as master–master or active/active replication. In this setup, more than one node is empowered to accept writes. Despite multiple nodes being capable of processing writes, the fundamental concept of replication remains unchanged: each node handling a write must disseminate that data change to all other nodes in the configuration.

#### Key Points:

1. **Leader Redundancy:**
   - Overcomes the single point of failure inherent in a single-leader model.
   - If one leader becomes unreachable, another leader can still accept writes.

2. **Data Change Propagation:**
   - Every node processing a write ensures the propagation of that data change to all other nodes.

### Considerations:

- **Consistency Challenges:**
  - Achieving consistency across multiple leaders can introduce complexities and challenges.

- **Conflict Resolution:**
  - Handling conflicts in scenarios where multiple leaders process writes simultaneously requires careful consideration.

### Advantages:

- **Improved Fault Tolerance:**
  - Overcomes the limitation of a single leader for write acceptance, enhancing fault tolerance.

- **High Availability:**
  - Enables continued write operations even if one leader becomes inaccessible.

### Insight:

Multi-leader replication addresses the single leader's limitations by distributing the ability to accept writes across multiple nodes. While introducing fault tolerance and high availability, it also brings challenges related to consistency and conflict resolution that need thoughtful management.