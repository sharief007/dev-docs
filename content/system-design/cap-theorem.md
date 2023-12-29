---
title: 'CAP Theorem'
weight: 1
type: docs
toc: true
prev: 
next: 
params:
  editURL:
---

The CAP theorem, proposed by Eric Brewer, states that in a distributed system, you can achieve at most two out of the three properties: Consistency (C), Availability (A), and Partition Tolerance (P).

According to the CAP theorem, when a network partition happens, you need to make a trade-off between Consistency and Availability.

#### Network Partition

A network partition is a situation where communication between different nodes or components of the distributed system is either slow or completely lost. It might be due to network failures, latency, or other issues.

#### Centralized System 

In a centralized systems(e.g., RDBMS), there is no concept of network partitions. All components of the system are within a single network, and communication between different parts of the system is assumed to be reliable and instantaneous. Hence, you effectively get both Availability and Consistency. 

#### Distributed System

In a distributed system, network partitions can occur. This means that in the face of a network partition, you have to choose whether to prioritize consistency of data (making sure all nodes see the same data) or availability of the system (ensuring that every request receives a response without guaranteeing that it contains the most recent version of the data).

 - If you choose Consistency, it might mean sacrificing Availability during a partition.
 - If you choose Availability, it might mean sacrificing Consistency, allowing nodes to diverge in their views of the data during a partition.

 - In CAP terms:
    - **CA:** Consistent and Available, but not Partition Tolerant (not suitable for distributed systems).
    - **CP:** Consistent and Partition Tolerant, sacrificing Availability during a partition.
    - **AP:** Available and Partition Tolerant, sacrificing Consistency during a partition.

Consistency Patterns: [Consistency Models](/dev-docs/system-design/database/replication/consistency-models/)  
Availability Patterns: [Replication](/dev-docs/system-design/database/replication/)