---
title: Messaging & Event-Driven
weight: 7
type: docs
toc: false
---

{{< cards >}}
  {{< card link="queues-vs-streams" title="Message Queues vs Event Streams" subtitle="RabbitMQ vs Kafka, work distribution vs fan-out, replay, consumer groups, and hybrid patterns" >}}
  {{< card link="kafka" title="Kafka Deep Dive" subtitle="Broker architecture, partitions, ISR, consumer groups, offset management, delivery guarantees, and storage internals" >}}
  {{< card link="event-driven-architecture" title="Event-Driven Architecture" subtitle="Events vs commands, pub/sub, event sourcing, CQRS, schema evolution, and debugging async flows" >}}
  {{< card link="cqrs" title="CQRS" subtitle="Separate write & read models, projections, sync strategies, eventual consistency handling, and when to use" >}}
  {{< card link="dlq-and-retry" title="Dead Letter Queues & Retry Strategies" subtitle="Exponential backoff with jitter, poison message handling, DLQ monitoring, and reprocessing workflows" >}}
{{< /cards >}}
