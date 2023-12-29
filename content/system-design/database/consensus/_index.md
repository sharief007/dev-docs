---
title: 'Consensus'
type: docs
toc: false
sidebar:
  open: true
prev: 
next:
params:
  editURL:
---

Consensus in distributed computing is a crucial and fundamental problem, aiming to achieve agreement among multiple nodes. At its core, the objective is to ensure that several nodes reach a unanimous decision or agreement on a certain matter. Despite its seemingly straightforward goal, the challenge of achieving consensus has proven to be complex.

In various scenarios, achieving agreement among nodes is crucial, such as:

1. **Leader Election:** 
- In a database employing single-leader replication, nodes must unanimously agree on the leader. 
- Without consensus, a split brain situation could occur, with two nodes both assuming leadership. This situation risks data divergence, inconsistency, and potential data loss.

2. **Atomic Commit:** 
- Databases supporting transactions across multiple nodes face challenges when a transaction fails on some nodes but succeeds on others.
- Consensus becomes essential to decide the transaction outcome: either all nodes commit (if all goes well) or all abort/roll back (if any issues arise).
- This consensus scenario is commonly known as the atomic commit problem.