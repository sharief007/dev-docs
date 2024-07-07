---
title: 'Database Anomalies'
weight: 1
type: docs
toc: true
prev: 
next: isolation-levels
params:
  editURL:
---


### 1. Dirty Reads

**Definition:**
A dirty read happens when a transaction reads data that has been modified by another transaction but not yet committed. If the other transaction rolls back, the data read becomes invalid.

```mermaid
sequenceDiagram
    participant A as Transaction A
    participant B as Transaction B
    participant DB as Database

    A->>DB: Update balance to $150
    B->>DB: Read balance (sees $150)
    A->>DB: Rollback (balance back to $100)
    B->>B: Sees dirty read ($150)

```


### 2. Unrepeatable Reads

**Definition:**
An unrepeatable read happens when a transaction reads the same data twice but gets different values each time because another transaction has modified the data in between.

```mermaid
sequenceDiagram
    participant A as Transaction A
    participant B as Transaction B
    participant DB as Database

    A->>DB: Read balance ($100)
    B->>DB: Update balance to $150 and commit
    A->>DB: Read balance again ($150)
    A->>A: Unrepeatable read ($100 to $150)

```


### 3. Phantom Reads

**Definition:**
A phantom read occurs when a transaction executes a query that returns a set of rows satisfying a condition, but a subsequent execution of the same query returns a different set of rows because another transaction inserted or deleted rows.

```mermaid
sequenceDiagram
    participant A as Transaction A
    participant B as Transaction B
    participant DB as Database

    A->>DB: Count customers with balance > $100 (finds 10)
    B->>DB: Insert new customer with balance $120 and commit
    A->>DB: Count customers with balance > $100 again (finds 11)
    A->>A: Phantom read (10 to 11)
```