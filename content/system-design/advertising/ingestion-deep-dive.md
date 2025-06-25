---
title: 'Ingestion Deep Dive'
weight: 4
toc: false
sidebar:
  open: false
prev: high-level-design
next: streaming-deep-dive
params:
  editURL: 
---

### 1. Push-Based Model: Direct to Broker vs. Intermediate Service

It's **generally okay and often preferred to directly send messages from lightweight agents (like Fluentd, Filebeat) to a robust message broker like Kafka.** This is the standard, battle-tested pattern for high-throughput log/event ingestion.

Here's why and when an intermediate service might be considered:

#### 1.1 Direct Agent-to-Broker Communication (Recommended)

* **Simplicity and Efficiency:** Removing an intermediate service reduces hops, latency, and points of failure. Agents are designed to be highly efficient at sending data directly to message queues.
* **Scalability:** Both the agents and Kafka are designed for massive horizontal scalability. Adding an intermediate service simply adds another layer that needs to scale, complicate deployment, and could become a bottleneck.
* **Reliability:** Modern agents have built-in retry mechanisms, local buffering, and acknowledgment protocols to ensure "at-least-once" delivery, even if the broker is temporarily unavailable. Kafka itself provides strong durability guarantees.
* **Authentication/Authorization:** Kafka supports various authentication mechanisms (e.g., SASL) and authorization (ACLs). Agents can be configured with credentials to securely connect to Kafka.
* **Centralized Configuration/Management:** While agents are distributed, their configurations (what logs to tail, where to send them) can be managed centrally using configuration management tools (Ansible, Puppet, Chef, Kubernetes ConfigMaps) or specialized agent management platforms.
* **Data Format Flexibility:** Agents can often be configured to parse, transform, and enrich data before sending it, providing some "centralization" of data formatting logic without a separate service.

#### 1.2 When to Consider an Intermediate Service (API Gateway / Ingestion Service)

An intermediate service (often called an **Ingestion Service** or **API Gateway**) between the agents and the message broker is primarily useful for:

1.  **Complex Authentication/Authorization:** If your agents are running in highly untrusted environments, or if you need a very granular, dynamic, or custom authentication/authorization logic (e.g., tied to an internal identity provider) that Kafka's native security features don't fully cover.
2.  **Protocol Translation/Standardization:** If you have many different types of agents or clients sending data in various formats/protocols, and you want to normalize them into a single, standardized format *before* hitting Kafka. For example, if some clients send HTTP POSTs, others use a custom binary protocol, and you want them all to become Avro messages in Kafka.
3.  **Client-Side Rate Limiting / Quotas:** To enforce strict rate limits or quotas on a per-client basis *before* data even hits your message broker, protecting it from abusive clients. Kafka also has its own producer quotas, but an upstream service can provide finer-grained control.
4.  **Complex Pre-Processing / Enrichment:** If you need to perform significant data enrichment or validation *before* data enters the streaming pipeline, and this logic is too heavy or too centralized for individual agents. Examples: looking up geo-IP information, cross-referencing with a customer database, or complex schema validation.
5.  **Centralized Error Handling/Dead-Lettering for Ingestion:** A single service to catch malformed events or authentication failures at the ingestion point and route them to a dedicated error queue or alert system.

**Decision for this Ad Click System:**

Adding an intermediate service would introduce more latency, more operational complexity (another distributed service to manage), and potential bottlenecks without a clear, overwhelming benefit for this specific use case.

### 2. Push vs. Pull Model for Log Ingestion

#### Push Model vs. Pull Model for Log Ingestion: A Comparison

| Feature             | Push Model (e.g., Filebeat/Fluentd to Kafka)                                       | Pull Model (e.g., Central Collector via SSH/API)                                 | Rationale for Ad Click System             |
| :------------------ | :--------------------------------------------------------------------------------- | :------------------------------------------------------------------------------- | :---------------------------------------- |
| **Mechanism** | Agents on source servers detect new logs and **push** them to a central destination. | A central collector **pulls** logs from source servers periodically.           | **Push:** Needed for real-time aggregation. |
| **Latency** | **Low/Near Real-time**: Events sent immediately upon generation.                  | **Higher**: Depends on polling interval; inherent delays in periodic pulls.      | **Push:** Critical for "few minutes" latency requirement. |
| **Scalability (Ingestion)** | **Highly Scalable**: Work distributed across many agents; horizontal scaling of message queue. | **Potential Bottleneck**: Central collector can become overloaded with many sources. | **Push:** Handles 1B clicks/day and 30% YoY growth much better. |
| **Resource Distribution** | **Distributed**: Agents consume resources on source servers.                     | **Centralized**: Collector consumes significant resources (CPU, Network, Disk).   | **Push:** Spreads load, avoids a single point of congestion. |
| **Reliability** | **High**: Agents buffer locally; "at-least-once" delivery with retries to message queue. | **Lower**: If collector fails, no logs collected until recovery; difficult to buffer on source. | **Push:** Minimizes data loss, crucial for billing accuracy. |
| **Deployment/Management** | Requires deploying and managing agents on all source servers.                      | No agents on source servers, but complex central collector setup.               | **Push:** Agent management is manageable with automation tools (e.g., Ansible). |
| **Network Overhead**| Continuous, but often optimized for streaming; agents can batch intelligently.      | Can be high during polling periods, as collector initiates many connections.      | **Push:** More efficient for continuous, high-volume streams. |
| **Security** | Agents have limited permissions (read logs); Kafka supports auth/authz.              | Central collector needs credentials/access to all source servers (higher risk).   | **Push:** Generally better security posture by distributing access. |
| **Data Integrity** | Stronger guarantees with agent buffering and message queue durability.              | Susceptible to gaps if collector experiences issues or network problems.          | **Push:** Essential for data used in billing. |
| **Flexibility** | Agents can perform basic parsing/filtering before sending.                          | Centralized parsing/filtering, but might be too late for some scenarios.        | **Push:** Allows early, lightweight transformations closer to source. |
| **Use Cases** | Real-time monitoring, analytics, security event pipelines.                         | Archival logging, less time-sensitive auditing, small to medium scale.         | **Push:** Clearly aligns with real-time ad click analytics. |

**Decision for this Ad Click System:**

For an ad click system with **1 billion events per day, real-time aggregation, and a few minutes of end-to-end latency**, the **push model is overwhelmingly superior and necessary.**

### 3. Explaining Fluentd, Filebeat, Logstash Shippers

These are all popular lightweight agents (or components of agents) used for log collection. They belong to the "push model" paradigm.

#### 3.1. Filebeat

* **Origin:** Part of the Elastic Stack (Elasticsearch, Kibana, Beats).
* **Purpose:** A lightweight, single-purpose data shipper. It specifically focuses on forwarding log files (tailing files).
* **Key Features:**
    * **Lightweight:** Written in Go, uses minimal resources.
    * **Reliable:** Guarantees "at-least-once" delivery by tracking file offsets and using acknowledgments. Can buffer locally.
    * **Simple Configuration:** Easy to set up using YAML files.
    * **Modules:** Pre-built modules for common log types (e.g., Apache, Nginx, MySQL) that automatically discover log paths and apply parsing.
    * **Output:** Can send to Elasticsearch, Logstash, Kafka, Redis, etc.
* **Use Case in Ad Click System:** Excellent choice for directly tailing the ad click log files and sending them to Kafka. Its low resource footprint makes it ideal for running on production ad servers.

#### 3.2. Fluentd

* **Origin:** Open-source project by Treasure Data, designed as a unified logging layer.
* **Purpose:** A versatile data collector for "unified logging" – collecting, parsing, transforming, and outputting log data from various sources to various destinations.
* **Key Features:**
    * **Plugin-based Architecture:** Highly extensible with over 500 plugins for input sources, output destinations, parsers, filters, and buffers.
    * **Flexible Data Transformation:** Can parse complex log formats, filter events, enrich data (e.g., add host metadata), and reformat messages.
    * **Reliable Buffering:** Supports memory and file buffering for reliability.
    * **Performance:** Written primarily in C and Ruby, designed for high throughput.
    * **Unified Logging:** Aims to standardize log data into structured JSON, making it easier for downstream processing.
* **Use Case in Ad Click System:** Could be used if more complex on-server parsing, filtering, or minor enrichment is needed before sending to Kafka. For example, if the log format is highly unstructured and needs significant regex parsing. It's more powerful than Filebeat but also slightly more resource-intensive.

#### 3.3. Logstash Shippers (Legacy/Niche)

* **Origin:** Part of the Elastic Stack.
* **Purpose:** Historically, Logstash was used in two modes: a central log processing pipeline and a lightweight "shipper" on individual servers.
* **Key Features (as a shipper):** Similar to Filebeat, but Logstash is built on Java/JRuby and consumes significantly more resources.
* **Modern Context:** **Filebeat has largely replaced Logstash as the preferred lightweight shipper.** Logstash is now primarily used as a central processing pipeline for more complex transformations, filtering, and routing of data *after* it's been collected by a lightweight shipper like Filebeat.
* **Use Case in Ad Click System:** **Not recommended as a shipper on ad servers due to its higher resource consumption.** Filebeat or Fluentd are better choices. Logstash could be used downstream if complex routing or transformations are needed *before* the stream processing layer, but this would add latency and complexity.

**In summary:** For tailing log files and pushing them to Kafka, **Filebeat** is generally the top recommendation due to its lightweight nature and reliability. **Fluentd** is a strong contender if more intricate on-server processing or a highly plugin-based approach is desired. Logstash is typically relegated to a central processing role, not an edge-agent role.