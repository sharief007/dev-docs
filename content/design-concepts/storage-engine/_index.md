---
title: 'Database Index'
weight: 1
type: docs
toc: true
sidebar:
  open: true
prev: 
next:
params:
  editURL:
---

### What is an Index?

- **Definition:** An index is a data structure that improves the speed of data retrieval operations on a database table at the cost of additional writes and storage space.
- **Types:** Common types of indexes include B-tree indexes, hash indexes, bitmap indexes, and full-text indexes.
- **Primary Data vs. Indexes:** The primary data in a database is stored in tables. An index creates an additional data structure that holds a sorted or otherwise organized reference to the data in the table.

### Benefits of Indexes

1. **Speed Up Reads:**
   - **Faster Search:** Indexes allow the database to quickly locate the data without scanning the entire table. For example, a B-tree index on a column allows binary search, reducing the time complexity of lookups from O(n) to O(log n).
   - **Efficient Sorting:** Indexes can also speed up sorting operations, as the index itself is often sorted.

2. **Optimized Queries:**
   - **Selective Queries:** Queries that filter on indexed columns can retrieve results faster because the index narrows down the data before accessing the table.

### Drawbacks of Indexes

1. **Slower Writes:**
   - **Insertions, Updates, and Deletions:** Every time a write operation is performed on the table, the index must also be updated. This incurs additional overhead, making write operations slower.
   - **Maintenance Cost:** Keeping the index up-to-date with the latest table data involves additional processing, especially if the index is complex or there are many indexes.

2. **Storage Overhead:**
   - **Additional Space:** Indexes require extra storage space. For large tables, the index itself can become substantial in size.

### Trade-Off in Practice

- **Read-Heavy Workloads:** If the application primarily performs read operations (e.g., search engines, reporting systems), using more indexes can significantly enhance performance.
- **Write-Heavy Workloads:** If the application performs frequent write operations (e.g., logging systems, real-time data collection), using fewer indexes can improve write performance by reducing the overhead.