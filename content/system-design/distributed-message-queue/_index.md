---
title: 'Distributed Message Queue'
weight: 1
type: docs
---

You are asked to design a **distributed, durable message queue / event streaming platform** like Apache Kafka. Producers publish messages to named **topics**; consumers read them, potentially long after they were written, and potentially many times by different independent applications. The platform is the central nervous system of a modern architecture: it decouples services, absorbs load spikes, feeds stream processors, and serves as a durable commit log for event-driven and event-sourced systems.

The design tension is delivering **high throughput** (millions of messages/second) with **durability** (never lose an acknowledged message) and **ordering** guarantees, while allowing many independent consumers to read at their own pace — all on commodity machines that fail. The core idea that makes this tractable is deceptively simple: a topic is a **partitioned, append-only log** on disk.

## Functional Requirements

1. **Publish:** a producer sends a message (optionally with a key) to a topic; it is durably stored.
2. **Subscribe / consume:** a consumer reads messages from a topic in order, tracking its position (offset).
3. **Consumer groups:** multiple consumers share a topic's partitions for horizontal scale; each partition is consumed by exactly one member of a group.
4. **Retention:** messages are retained for a configured time or size, independent of consumption.
5. **Replication:** each partition is replicated across brokers so a broker failure loses no acknowledged data.
6. **Delivery guarantees:** at-least-once by default; at-most-once and exactly-once as options.

## Out of Scope

- Complex message routing/transformations (that's a stream-processing layer on top).
- Per-message TTL / priority queues / delayed delivery (classic broker features, not the Kafka model).
- The client applications' business logic.

## Non-Functional Requirements

- **Throughput:** 1M+ messages/s aggregate write; multiples of that on read (multiple consumer groups). Sustained GB/s.
- **Durability:** an **acked** message survives f broker failures (configurable, typically f=2 via 3 replicas). Zero data loss for acked writes.
- **Latency:** produce-to-ack p99 **< 10 ms** for in-memory-page-cache writes; end-to-end consume lag seconds or less.
- **Ordering:** **total order within a partition**; no global order across partitions (by design).
- **Availability:** **99.95%**; a partition stays writable as long as a quorum of its replicas is alive.
- **Scalability:** add brokers to add capacity linearly; a topic scales by adding partitions.
