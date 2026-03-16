---
title: Consistent Hashing
weight: 11
type: docs
toc: true
sidebar:
  open: true
prev:
next:
params:
  editURL:
---

Consistent hashing solves a specific problem: how to distribute keys across a dynamic cluster of nodes such that adding or removing a node moves the minimum possible number of keys. It is the foundation of distributed caches, Cassandra's storage layer, and any system where the set of servers is not fixed.

## The Problem with Naive Modulo Hashing

The intuitive approach to distributing keys across N nodes is:

```
node = hash(key) % N
```

This works perfectly when N never changes. When N changes — a node fails, or you add capacity — it breaks catastrophically.

**Why?** Each key's node assignment is `hash(key) % N`. When N changes to N+1, `% N` and `% (N+1)` produce different results for almost all keys.

```
N = 3 nodes. hash(key) % 3:

key_A → hash=10 → 10%3 = 1 → Node 1
key_B → hash=11 → 11%3 = 2 → Node 2
key_C → hash=12 → 12%3 = 0 → Node 0
key_D → hash=13 → 13%3 = 1 → Node 1

Add 1 node. N = 4. hash(key) % 4:

key_A → hash=10 → 10%4 = 2 → Node 2  ← MOVED
key_B → hash=11 → 11%4 = 3 → Node 3  ← MOVED
key_C → hash=12 → 12%4 = 0 → Node 0  ← same
key_D → hash=13 → 13%4 = 1 → Node 1  ← same
```

In this example, 2 of 4 keys moved. With uniformly distributed hashes and N nodes, adding one node remaps approximately **(N−1) / N** of all keys to different nodes:

| N (before) | Fraction of keys remapped |
|-----------|--------------------------|
| 3 → 4 | ~75% |
| 9 → 10 | ~90% |
| 99 → 100 | ~99% |

For a distributed cache with millions of keys, a 90% remapping rate on a scale-up event means a near-complete cache invalidation — every remapped key becomes a cache miss that hits the origin simultaneously. This is a **thundering herd** that can take down the origin.

Consistent hashing reduces the fraction of keys remapped to approximately **1 / (N+1)** — only the keys that belong to the new node's portion of the key space.

## The Hash Ring

Map the hash output space onto a circle (ring). Both nodes and keys are hashed to positions on the ring.

```
Hash space: [0, 2^32)  arranged as a ring

                    0
                 ●
              /     \
      Node A         Node C
        ●               ●
        |               |
      2^32/4          3*2^32/4
        |               |
      Node B
        ●
            \       /
              2^32/2
```

**Node placement:** Hash each node's identifier (e.g., IP address or `node_name`) to a position on the ring.

**Key lookup rule:** To find which node owns a key, hash the key to a ring position and walk **clockwise** until you find a node. That node owns the key.

```
Ring positions (simplified to 0–99):

  Node A: position 10
  Node B: position 40
  Node C: position 70

Key "foo" → hash=25 → walk clockwise → hits Node B at 40 → owned by Node B
Key "bar" → hash=55 → walk clockwise → hits Node C at 70 → owned by Node C
Key "baz" → hash=85 → walk clockwise → wraps around → hits Node A at 10 → owned by Node A
```

**Node arc ownership:** Each node owns the arc of the ring between itself and the previous node (counter-clockwise). Node B at position 40 owns keys that hash to positions [11, 40].

## Adding and Removing Nodes

### Adding a Node

New node D is placed at position 55 on the ring.

```
Before:                          After adding Node D at 55:
  ...Node B(40)...Node C(70)...    ...Node B(40)...Node D(55)...Node C(70)...

Keys previously owned by Node C  →  Keys in arc [41, 55] now owned by Node D
  that hash to [41, 55]              Keys in arc [56, 70] remain with Node C
```

Only keys in the arc [41, 55] need to move — from Node C to Node D. No other keys are affected. With N existing nodes of equal size, the new node takes approximately 1/(N+1) of the total keys. For a 10-node cluster, adding one node moves ~9% of keys, not 90%.

### Removing a Node (or Node Failure)

Node B at position 40 is removed.

```
Before: ...Node A(10)...Node B(40)...Node C(70)...
After:  ...Node A(10)...Node C(70)...

Keys owned by Node B [11, 40] → reassigned to Node C (next clockwise node)
```

Only keys that were on Node B move. All other keys are unaffected.

## Virtual Nodes

Plain consistent hashing with one ring position per physical node has a critical flaw: **non-uniform distribution**.

With 3 nodes, the arcs are not guaranteed to be equal. If Node A lands at position 10, Node B at 11, and Node C at 90, then:
- Node A owns arc [91, 10] = 20 positions (20% of keys)
- Node B owns arc [11, 11] = 1 position (1% of keys) ← massively under-loaded
- Node C owns arc [12, 90] = 79 positions (79% of keys) ← massively over-loaded

Real hash functions produce roughly uniform positions across many nodes, but with small clusters the variance is high.

**Virtual nodes (vnodes):** Each physical node is assigned **multiple positions** on the ring instead of one. Each position is a virtual node — a separate hash of the physical node identity.

```
Physical node A → Virtual positions: A_1(8), A_2(35), A_3(67), A_4(89)
Physical node B → Virtual positions: B_1(22), B_2(48), B_3(72), B_4(95)
Physical node C → Virtual positions: C_1(14), C_2(55), C_3(80), C_4(99)

Ring (positions 0–99):
 8   14  22  35  48  55  67  72  80  89  95  99
[A1][C1][B1][A2][B2][C2][A3][B3][C3][A4][B4][C4]
```

**Benefits of vnodes:**

| Benefit | Why |
|---------|-----|
| **Even distribution** | With enough vnodes (128–256 per node), statistical variance in load averages out |
| **Smoother rebalancing** | When a node is added, it takes small arcs from many existing nodes — load spreads evenly rather than burdening one neighbor |
| **Heterogeneous hardware** | Assign more vnodes to more powerful machines — they own more arcs and more keys proportionally |
| **Faster recovery** | A failed node's keys are redistributed to many nodes (each taking one small arc) rather than one successor node taking all the load |

**Vnode count tradeoff:** More vnodes = better distribution, but also larger routing metadata (each node must know all other nodes' vnode positions). Cassandra defaults to 256 vnodes per node. For very large clusters (1000+ nodes), 8–16 vnodes is a pragmatic compromise.

## Replication on the Ring

In a distributed storage system, each key is replicated to R nodes for fault tolerance. The standard approach: store the key on the **R successive nodes clockwise** from the key's position.

```
Replication factor R = 3:

key "foo" → position 25 → clockwise: Node B(40), Node C(70), Node A(10+wrap)
                                      ↑primary   ↑replica1  ↑replica2
```

**Why this interacts well with consistent hashing:** When Node B fails, Node C (already a replica) promotes to primary for those keys. The system reads from the next available replica with no data loss — the replica was already there.

With vnodes: the R successive clockwise virtual nodes must belong to R **different physical nodes** — otherwise a single machine failure loses all replicas. Cassandra enforces this by skipping vnodes that belong to a physical node already in the replica set.

## Real-World Applications

### Cassandra Token Ring

Cassandra uses consistent hashing as its core partitioning mechanism. The ring positions are called **tokens**. See [Wide-Column Stores](../wide-column-stores) for how Cassandra builds its full read and write path on top of this ring.

```
Row key → Murmur3 hash → token (64-bit integer)
Token → mapped to the node that owns that token range
```

In Cassandra 3.0+, vnodes (called "token ranges") are the default with 256 vnodes per node. The coordinator node (the node that receives a client request) routes the request to the primary replica using the ring, then waits for quorum acknowledgement from the replicas.

### Memcached (libketama)

Consistent hashing for Memcached was popularized by libketama (Last.fm, 2007), which added client-side consistent hashing to Memcached's simpler modulo-based approach. The client hashes both the server list and each key to a ring, routing each key to the correct server without a central coordinator.

### Load Balancer Session Affinity

Some L7 load balancers use consistent hashing on a client attribute (source IP, session cookie, user ID) to route requests from the same client to the same backend server. Unlike IP hash (which breaks when backends are added), consistent hashing moves only 1/N of existing sessions on a scale event — most sessions stay with their current backend.

### Redis Cluster (Hash Slots — a related approach)

Redis Cluster uses **16,384 fixed hash slots** rather than a continuous ring:

```
slot = CRC16(key) % 16384
slot 0–5460    → Node A
slot 5461–10922 → Node B
slot 10923–16383 → Node C
```

This is conceptually similar to virtual nodes on a ring but with a fixed discrete number of slots. When resharding, individual slots are moved between nodes. The slot count (16,384) is chosen to allow fine-grained redistribution while keeping the slot map small enough to gossip between nodes (2 KB bitmap).

## Hotspot Mitigation

Virtual nodes reduce statistical load imbalance, but they cannot solve **application-level hotspots** — a single key receiving disproportionate traffic.

**Example:** A viral tweet with tweet_id=99 receives 1 million requests per second. Consistent hashing assigns it to one node. Every request for that tweet hits the same node regardless of how many vnodes exist — vnodes distribute different keys, not traffic for the same key.

**Strategies:**

| Strategy | How | When to use |
|----------|-----|-------------|
| **Application-level cache** | Cache the hot object in local in-process memory — requests never reach the distributed cache | Hot objects known ahead of time (product launches, featured content) |
| **Key replication with random suffix** | Store the hot key as N copies: `tweet:99:shard_0` through `tweet:99:shard_7` — reads pick a random shard | Hot keys not predictable; read-heavy |
| **Explicit shard splitting** | Manually assign a hot partition to its own dedicated node | Known-hot partitions in Cassandra (override token assignment) |
| **Read replicas with scatter** | Direct reads for hot keys to replicas; LB distributes across R replicas | Works with consistent hashing replication |
| **CDN / edge cache** | Cache the hot object at CDN edge nodes — requests never reach the origin cluster | Public, cacheable content |

**The fundamental limit:** Consistent hashing distributes a keyspace across nodes. It cannot distribute a single key's traffic across nodes — that requires application-level routing logic above the hashing layer.

## Summary: Why Consistent Hashing Works

| Property | Naive mod hashing | Consistent hashing |
|----------|------------------|-------------------|
| Keys remapped on node add | ~(N-1)/N ≈ 90% for large N | ~1/(N+1) ≈ 9% for N=10 |
| Keys remapped on node remove | ~(N-1)/N ≈ 90% | Only keys on removed node (~1/N) |
| Load distribution | Uniform (by math) | Skewed without vnodes; uniform with vnodes |
| Hot key handling | Same problem | Same problem — needs application layer |
| Complexity | O(1) lookup | O(log N) lookup (binary search on sorted ring) |
