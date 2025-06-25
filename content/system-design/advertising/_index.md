---
title: 'Ad Click Event Aggregation'
weight: 1
toc: true
sidebar:
  open: true
prev: 
next: digital-ads-terminology
params:
  editURL: 
---


Imagine you are tasked with designing a real-time system to aggregate and analyze ad click events for a large advertising platform. This system needs to handle massive data volumes, provide near real-time insights for reporting and billing, and be resilient to various failures.

**Input Data:**

Click events are appended to log files located on various ad servers. Each click event has the following attributes:

* `click_timestamp`: When the click occurred (Unix epoch, milliseconds).
* `user_id`: Unique identifier for the user.
* `ad_id`: Unique identifier for the advertisement clicked.
* `ip`: IP address of the user.
* `country`: Country of the user.

**Data Volume:**

* **Ingestion Rate:** Approximately **1 billion ad clicks per day**.
* **Total Ads:** Roughly **2 million unique ads** in the system.
* **Growth:** The number of ad click events is projected to grow by **30% year-over-year**.

**Functional Requirements:**

The system must support the following primary queries and aggregation capabilities:

1.  **Ad Click Count (Per Ad):**
    * Return the total number of click events for a *particular `ad_id`* within the last `M` minutes.
    * `M` should be a configurable parameter (e.g., 1, 5, 10, 60 minutes).
2.  **Top N Most Clicked Ads:**
    * Return the **top 100 most clicked `ad_id`s** in the past `M` minutes.
    * `M` should be a configurable parameter (e.g., 1, 5, 10, 60 minutes).
    * Aggregation for this query should occur and be updated every minute.
3.  **Filtered Aggregations:**
    * Both of the above queries (Ad Click Count and Top N Ads) must support filtering by `ip`, `user_id`, or `country`. This means a user should be able to ask for, for example, "top 100 ads from users in the USA in the last 5 minutes."

**Non-Functional Requirements:**

1.  **Latency:**
    * End-to-end latency (from click event generation to queryable aggregation) should be **within a few minutes**.
    * Note: This latency is acceptable as the primary use cases are ad billing and reporting, not real-time bidding (RTB) which has much stricter sub-second latency requirements.
2.  **Correctness:**
    * The accuracy of aggregation results is paramount, as the data is used for billing and critical reporting.
3.  **Scalability:**
    * The system must be horizontally scalable to handle current data volumes and the projected 30% YoY growth, effectively operating at "Facebook or Google scale" for this specific use case.
4.  **Reliability & Resilience:** The system must account for and gracefully handle the following edge cases:
    * **Late Events:** Click events might arrive later than their `click_timestamp` due to network delays or system issues.
    * **Duplicated Events:** The same click event might be sent multiple times by upstream systems or agents.
    * **System Failures:** Individual components or parts of the system might go down at any time, requiring robust recovery mechanisms without data loss.
