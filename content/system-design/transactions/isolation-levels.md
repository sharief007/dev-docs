---
title: 'Isolation levels'
weight: 3
type: docs
toc: true
prev: 
next: 
params:
  editURL:
---

Transaction isolation levels define the degree to which the operations in one transaction are isolated from those in other transactions. 
### 1. Read Uncommitted

**Description:**
- The lowest isolation level.
- Transactions can see uncommitted changes made by other transactions (allows dirty reads).

**How It’s Achieved:**
- No locks are placed on the data.
- Minimal overhead as the system does not enforce isolation.

### 2. Read Committed

**Description:**
- Transactions can only see changes made by other transactions once those changes are committed.
- Prevents dirty reads but allows unrepeatable reads and phantom reads.

**How It’s Achieved:**
- Shared locks are placed on data being read, which are released immediately after the read.
- Write locks are held until the transaction is committed or rolled back.

### 3. Repeatable Read

**Description:**
- Transactions can read the same data multiple times and get the same value each time (prevents unrepeatable reads).
- Does not prevent phantom reads.

**How It’s Achieved:**
- Shared locks are placed on all data read by the transaction and are held until the transaction completes.
- Write locks are also held until the transaction completes.
- Other transactions cannot modify the data being read until the current transaction is complete.

### 4. Serializable

**Description:**
- The highest isolation level.
- Ensures complete isolation from other transactions.
- Guarantees that transactions are executed in a serial (non-concurrent) manner.

**How It’s Achieved:**
- Range locks are placed on data sets to prevent other transactions from inserting, updating, or deleting rows that would affect the current transaction's results.
- This level uses locking and/or multiversion concurrency control (MVCC) to ensure strict isolation.


| Isolation Level  | Dirty Reads | Unrepeatable Reads | Phantom Reads | Mechanism                                      |
|------------------|-------------|--------------------|---------------|------------------------------------------------|
| Read Uncommitted | Yes         | Yes                | Yes           | No locks                                       |
| Read Committed   | No          | Yes                | Yes           | Shared/Exclusive locks                         |
| Repeatable Read  | No          | No                 | Yes           | Shared/Exclusive locks held till completion    |
| Serializable     | No          | No                 | No            | Range locks, MVCC, Timestamp ordering          |

### Behind the Scenes Mechanisms

1. **Locks:**
   - **Shared Locks (S):** Allow multiple transactions to read a resource but not modify it.
   - **Exclusive Locks (X):** Prevent any other transactions from reading or modifying the resource.
   - **Intent Locks:** Indicate a transaction's intention to acquire shared or exclusive locks, which helps coordinate between different types of locks.

2. **Multiversion Concurrency Control (MVCC):**
   - Instead of locking data, MVCC allows multiple versions of data to exist simultaneously.
   - When a transaction modifies data, a new version is created. Other transactions can continue to read the old version until the new version is committed.
   - This approach helps reduce contention and improve performance, particularly for read-heavy workloads.

3. **Range Locks:**
   - Used in the serializable isolation level to lock ranges of data, preventing other transactions from inserting, updating, or deleting rows within those ranges.

4. **Timestamp Ordering:**
   - Each transaction is given a unique timestamp.
   - The database ensures that transactions are executed in timestamp order, which helps prevent anomalies.

These isolation levels allow a balance between performance and data consistency, letting database administrators and developers choose the most appropriate level for their specific use cases.