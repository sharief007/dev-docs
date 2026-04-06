---
title: Reliability Patterns
weight: 9
type: docs
toc: false
sidebar:
  open: true
---

{{< cards >}}
  {{< card link="circuit-breaker" title="Circuit Breaker" subtitle="Closed/open/half-open states, rolling window failure detection, exponential backoff reset, fallback strategies, and composition with retries" >}}
  {{< card link="bulkhead" title="Bulkhead Pattern" subtitle="Thread pool and semaphore isolation, preventing cascading failures, and combining with circuit breakers" >}}
  {{< card link="back-pressure" title="Back-Pressure & Load Shedding" subtitle="TCP flow control, reactive streams, Kafka/RabbitMQ back-pressure, admission control, priority shedding, and CoDel" >}}
{{< /cards >}}