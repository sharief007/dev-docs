---
title: 'Replication Implementations'
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

### Statement-Based Replication:

In statement-based replication, the leader records and shares every executed write request (statement) with its followers. Each follower processes these statements, such as INSERT, UPDATE, or DELETE, similar to client-generated SQL statements.

#### Challenges:

**Nondeterministic Functions:** Statements calling functions like NOW() or RAND() may produce different results on each replica.

Workaround: The leader can replace nondeterministic function calls with a fixed return value in the statement log for consistency across followers.

**Side Effects from Statements:**  Statements with side effects (e.g., triggers, stored procedures) may generate varied results on different replicas. Resolving inconsistencies in side effects can be complex and may require specific handling.

Due to numerous edge cases and potential challenges in maintaining consistency, statement-based replication is often considered less preferable in favor of alternative replication methods.

### Write-Ahead Log (WAL) Shipping:

In scenarios involving structures like B-trees, where individual disk blocks are overwritten, modifications are initially recorded in a write-ahead log (WAL). This practice ensures that the index can be restored to a consistent state following a crash.

#### Replication Process:

**Log Shipping:**
- The leader not only writes the log to disk but also transmits it across the network to its followers.
- The follower, upon receiving the log, reconstructs identical data structures as those on the leader.

**Log Contents:**
- The WAL contains details about which bytes were altered in specific disk blocks.
- This close coupling with the storage engine makes replication intricately linked to the underlying storage format.

#### Limitations:

- WAL shipping, while efficient, closely ties replication to the storage engine. Careful consideration of version compatibility is crucial to maintain a consistent database state. 
- Leveraging this approach for zero-downtime upgrades requires a replication protocol that supports running different software versions on the leader and followers.


### Logical (Row-Based) Log Replication:

When specific requirements arise, such as replicating a subset of data, moving data between different types of databases, or implementing conflict resolution logic (refer to "Handling Write Conflicts"), it becomes necessary to elevate the replication process to the application layer. This shift allows for more customized and tailored solutions to address unique needs and scenarios.

#### Log Content:

**Inserted Row:** Log contains new values of all columns for the inserted row.

**Deleted Row:** Log provides sufficient information to uniquely identify the deleted row. If no primary key exists, old values of all columns need to be logged.

**Updated Row:** Log includes adequate details to uniquely identify the updated row, along with new values of all columns.

**Transaction Modification:** A transaction modifying multiple rows generates several log records. Followed by a record indicating the transaction's commitment.

#### Advantages:

**Backward Compatibility:** Logical log, being decoupled from storage engine internals, facilitates better backward compatibility. Allows the leader and follower to run different database software versions or even different storage engines.

**Ease of Parsing:** Logical log format is simpler for external applications to parse. Useful for sending database contents to external systems like data warehouses for offline analysis or building custom indexes and caches. This technique is called **change data capture**.

### Trigger-Based Replication:

In certain scenarios, where selective data replication, cross-database replication, or conflict resolution logic is required (refer to "Handling Write Conflicts"), it may be necessary to elevate replication to the application layer.

#### Tools and Techniques:

**Database Log Reading Tools:** Tools like Oracle GoldenGate can extract data changes from the database log and make them available to an application.

**Triggers and Stored Procedures:**
- Relational databases offer features like triggers and stored procedures.
- A trigger allows the registration of custom application code automatically executed on data change (write transaction).
- The trigger can log this change into a separate table, accessible for reading by an external process.

#### Considerations:

**Overheads and Limitations:** Trigger-based replication typically incurs higher overheads compared to other methods and may be more susceptible to bugs and limitations.

**Flexibility:** Despite potential drawbacks, the flexibility of trigger-based replication makes it valuable in specific use cases.

**Insight:**
Trigger-based replication, although more resource-intensive and potentially more error-prone than built-in database replication, offers unparalleled flexibility. It becomes essential in situations demanding custom logic, selective data replication, or replication across different database systems.