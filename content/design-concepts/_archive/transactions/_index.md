---
title: 'Transactions'
weight: 1
type: docs
toc: true
prev: 
next:
params:
  editURL:
---

### ACID Properties

**Atomicity**
- **Definition:** Ensures that all operations within a transaction are completed successfully. If any part of the transaction fails, the entire transaction fails and all changes are rolled back.
- **Why It's Important:** Without atomicity, partial changes could be saved, leading to an inconsistent state where some changes are applied and others are not. This makes it hard to know which changes have taken effect and which haven’t, potentially leading to duplicate or incorrect data if the transaction is retried.

**Consistency**
- **Definition:** Ensures that a transaction brings the database from one valid state to another, maintaining the defined rules (invariants) of the database.
- **Why It's Important:** Ensures the integrity of the database. Even if the application logic contains errors, the database enforces constraints (like foreign keys or unique keys) to prevent invalid data.

**Isolation**
- **Definition:** Ensures that transactions are executed in such a way that they do not affect each other, producing the same result as if they were run sequentially.
- **Why It's Important:** Prevents concurrency issues like race conditions, ensuring that transactions do not interfere with each other and the final state of the database is correct.

**Durability**
- **Definition:** Ensures that once a transaction has been committed, it will remain so, even in the event of a system crash.
- **Why It's Important:** Provides reliability in storing data, ensuring that once data is saved, it won’t be lost even if there are hardware or system failures.

### BASE Properties

**Basically Available**
- **Definition:** The system guarantees availability, meaning it will always give a response (though not necessarily immediately or correctly).

**Soft state**
- **Definition:** The state of the system may change over time, even without new input, due to eventual consistency.

**Eventual Consistency**
- **Definition:** The system guarantees that, given enough time, all replicas of the data will become consistent.

### Comparison: ACID vs. BASE

**ACID:**
- **Atomicity:** Ensures full completion or no completion of transactions.
- **Consistency:** Maintains database invariants after transactions.
- **Isolation:** Ensures transactions do not affect each other.
- **Durability:** Guarantees data is not lost after commit.

**BASE:**
- **Basically Available:** Prioritizes availability over immediate consistency.
- **Soft state:** Allows data to be in flux temporarily.
- **Eventual Consistency:** Guarantees that all nodes will see the same data eventually.

**Why Choose One Over the Other?**
- **ACID:** Ideal for applications where consistency and reliability are paramount (e.g., banking systems).
- **BASE:** Suitable for systems where availability and partition tolerance are more critical than immediate consistency (e.g., social media platforms).
